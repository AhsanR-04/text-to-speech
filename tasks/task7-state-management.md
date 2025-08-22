# Task 7: Implement State Management Pattern

## Priority: Medium 📋

## Description
Implement a centralized state management pattern to replace scattered state properties and improve application predictability and debugging.

## Current Issue
```javascript
// Scattered state properties
this.recognition = null;
this.isListening = false;
this.transcript = '';
this.interimTranscript = '';
this.language = 'en-US';
// ... more properties scattered throughout the class
```

State is scattered across multiple properties without centralized management or state change tracking.

## Solution
Implement a centralized state management system:

### State Manager Implementation
```javascript
class AppStateManager {
    constructor() {
        this.state = {
            // Recognition state
            recognition: null,
            isListening: false,
            isSupported: false,
            
            // Content state
            transcript: '',
            interimTranscript: '',
            characterCount: 0,
            
            // Configuration state
            language: 'en-US',
            autoScroll: true,
            autoSave: false,
            
            // UI state
            isVisible: true,
            notifications: [],
            currentError: null,
            
            // Performance state
            lastUpdateTime: null,
            updateCount: 0
        };
        
        this.listeners = new Map();
        this.history = [];
        this.maxHistorySize = 50;
    }
    
    // Get current state (immutable)
    getState() {
        return { ...this.state };
    }
    
    // Update state with validation and history
    setState(updates, source = 'unknown') {
        const previousState = { ...this.state };
        const newState = { ...this.state, ...updates };
        
        // Validate state changes
        if (this.validateStateChange(previousState, newState)) {
            this.state = newState;
            
            // Add to history
            this.addToHistory(previousState, newState, source);
            
            // Notify listeners
            this.notifyListeners(previousState, newState, updates);
        } else {
            console.warn('Invalid state change rejected:', updates);
        }
    }
    
    // Subscribe to state changes
    subscribe(callback, filter = null) {
        const id = Date.now() + Math.random();
        this.listeners.set(id, { callback, filter });
        return () => this.listeners.delete(id);
    }
    
    // Validate state changes
    validateStateChange(prevState, newState) {
        // Validate transcript length
        if (newState.transcript && newState.transcript.length > 50000) {
            return false;
        }
        
        // Validate language format
        if (newState.language && !/^[a-z]{2}-[A-Z]{2}$/.test(newState.language)) {
            return false;
        }
        
        // Validate boolean states
        const booleanFields = ['isListening', 'isSupported', 'autoScroll', 'autoSave'];
        for (const field of booleanFields) {
            if (newState[field] !== undefined && typeof newState[field] !== 'boolean') {
                return false;
            }
        }
        
        return true;
    }
    
    // Add state change to history
    addToHistory(prevState, newState, source) {
        this.history.push({
            timestamp: Date.now(),
            source,
            changes: this.getChanges(prevState, newState),
            prevState: { ...prevState },
            newState: { ...newState }
        });
        
        // Maintain history size
        if (this.history.length > this.maxHistorySize) {
            this.history.shift();
        }
    }
    
    // Get state differences
    getChanges(prevState, newState) {
        const changes = {};
        for (const key in newState) {
            if (prevState[key] !== newState[key]) {
                changes[key] = {
                    from: prevState[key],
                    to: newState[key]
                };
            }
        }
        return changes;
    }
    
    // Notify state change listeners
    notifyListeners(prevState, newState, updates) {
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
    
    // Check if updates match filter
    matchesFilter(updates, filter) {
        if (Array.isArray(filter)) {
            return filter.some(key => key in updates);
        }
        return filter in updates;
    }
    
    // Development helpers
    getHistory() {
        return [...this.history];
    }
    
    resetState() {
        this.setState({
            transcript: '',
            interimTranscript: '',
            characterCount: 0,
            isListening: false,
            currentError: null,
            notifications: []
        }, 'reset');
    }
}
```

### Integration with Main App
```javascript
class VoiceToTextApp {
    constructor() {
        this.stateManager = new AppStateManager();
        this.setupStateListeners();
        this.initializeApp();
    }
    
    setupStateListeners() {
        // Listen to transcript changes
        this.stateManager.subscribe((state, prevState) => {
            if (state.transcript !== prevState.transcript || 
                state.interimTranscript !== prevState.interimTranscript) {
                this.updateTranscriptDisplay();
            }
        }, ['transcript', 'interimTranscript']);
        
        // Listen to listening state changes
        this.stateManager.subscribe((state, prevState) => {
            if (state.isListening !== prevState.isListening) {
                this.updateListeningUI(state.isListening);
            }
        }, ['isListening']);
        
        // Listen to error state changes
        this.stateManager.subscribe((state, prevState) => {
            if (state.currentError !== prevState.currentError) {
                this.handleErrorState(state.currentError);
            }
        }, ['currentError']);
    }
    
    // Update methods to use state manager
    startListening() {
        this.stateManager.setState({
            isListening: true,
            currentError: null
        }, 'startListening');
        
        // ... recognition logic
    }
    
    stopListening() {
        this.stateManager.setState({
            isListening: false
        }, 'stopListening');
        
        // ... stop logic
    }
    
    addTranscript(text, isInterim = false) {
        const state = this.stateManager.getState();
        const updates = isInterim ? 
            { interimTranscript: text } : 
            { transcript: state.transcript + text, interimTranscript: '' };
            
        this.stateManager.setState(updates, 'addTranscript');
    }
}
```

## Acceptance Criteria
- [ ] All state is managed centrally
- [ ] State changes are validated
- [ ] State history is maintained for debugging
- [ ] Components can subscribe to specific state changes
- [ ] State changes are immutable
- [ ] Performance impact is minimal
- [ ] Development tools for state inspection

## Files to Modify
- `index.html` (add AppStateManager class)
- Update VoiceToTextApp to use state manager
- Refactor all state-related operations

## Benefits
- Predictable state management
- Better debugging capabilities
- Easier testing
- Centralized validation
- State change history
- Reduced state-related bugs

## Testing
1. Verify all state operations work correctly
2. Test state validation rules
3. Check state change notifications
4. Verify state history tracking
5. Test performance with frequent updates
6. Validate state consistency

## Estimated Time
2.5-3 hours
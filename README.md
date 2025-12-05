// Add debug for session ID (first 10 characters only for security)
System.debug('Session ID prefix: ' + (String.isNotBlank(sessionId) ? sessionId.substring(0, 10) + '...' : 'NULL'));

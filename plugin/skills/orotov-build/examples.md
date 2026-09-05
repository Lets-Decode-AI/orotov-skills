# OROTOV Build Examples

Two detailed examples showing the complete workflow from issue specification through implementation and review.

## Example 1: Simple Task - WebSocket Heartbeat

### Issue Specification

**Issue #12: Implement WebSocket ping/pong heartbeat**

**Description**: Add automatic heartbeat to keep WebSocket connection alive during idle periods.

**Acceptance Criteria**:
1. Sends ping every 30 seconds when idle
2. Stops sending when connection is active
3. Closes connection if pong not received within 5 seconds
4. Doesn't interfere with normal messages

**Test Requirements**:
- Test ping sent at 30-second intervals
- Test ping not sent when messages are flowing
- Test connection closed on pong timeout
- Test ping/pong messages don't interfere with normal traffic

**Story Context**: WebSockets can be dropped by proxies after inactivity. This feature keeps the connection alive without the application layer needing to send dummy data.

### Implementation Process

#### Step 1: Implementer Asks Clarifying Questions

"Looking at the spec, I have a few questions:

1. When you say 'doesn't interfere with normal messages', should we:
   - Reset the 30-second timer when we send/receive a real message?
   - Or send ping regardless of message traffic?

2. For 'pong not received within 5 seconds', should we:
   - Log the timeout?
   - Attempt to reconnect?
   - Or just close the connection?

3. Should ping/pong be configurable (e.g., 60 seconds instead of 30)?"

**Response**:
"Good questions. Yes, reset the timer on any message traffic (more typical). For timeout, just close and let the app handle reconnection. Keep hardcoded values for now (spec says 30s ping, 5s pong timeout)."

#### Step 2: Implementer Writes Tests

```python
import pytest
import time
from unittest import mock
from websocket_client import WebSocketClient

def test_heartbeat_sends_ping_every_30_seconds():
    """Test that ping is sent at 30-second intervals during idle."""
    with mock.patch('time.time') as mock_time:
        client = WebSocketClient()
        client.connect("ws://localhost:8000")
        
        # Simulate 30 seconds passing
        mock_time.return_value = 30
        client._check_heartbeat()
        
        # Verify ping was sent
        assert client._ws.send.called
        call_args = client._ws.send.call_args
        assert b'ping' in str(call_args).lower()

def test_heartbeat_resets_timer_on_message():
    """Test that the 30-second timer resets when messages are sent/received."""
    with mock.patch('time.time') as mock_time:
        client = WebSocketClient()
        client.connect("ws://localhost:8000")
        
        # Simulate 20 seconds passing
        mock_time.return_value = 20
        
        # Receive a message (should reset timer)
        client._handle_message("Hello")
        
        # Simulate 25 more seconds passing (45 total, but 25 since last message)
        mock_time.return_value = 45
        client._check_heartbeat()
        
        # Ping should not be sent yet (only 25 seconds since last message)
        assert not client._ws.send.called

def test_heartbeat_closes_connection_on_pong_timeout():
    """Test that connection closes if pong not received within 5 seconds."""
    with mock.patch('time.time') as mock_time:
        client = WebSocketClient()
        client.connect("ws://localhost:8000")
        
        # Send ping at 30 seconds
        mock_time.return_value = 30
        client._check_heartbeat()
        ping_sent_time = mock_time.return_value
        
        # 5+ seconds later, no pong received
        mock_time.return_value = ping_sent_time + 5.1
        client._check_heartbeat()
        
        # Connection should be closed
        assert client._ws.close.called

def test_heartbeat_doesnt_interfere_with_normal_messages():
    """Test that ping/pong messages don't interfere with normal message handling."""
    client = WebSocketClient()
    client.connect("ws://localhost:8000")
    
    # Mock the message handler
    handler = mock.Mock()
    client.on_message = handler
    
    # Send normal message
    client.send("Hello")
    
    # Simulate receiving normal message
    client._handle_message("World")
    
    # Verify normal message was handled
    assert handler.called
    assert handler.call_args[0][0] == "World"
    
    # Ping/pong should not trigger message handler
    client._handle_message("PING")
    client._handle_message("PONG")
    
    # Should not call handler twice (ping/pong filtered)
    assert handler.call_count == 1
```

#### Step 3: Implementer Implements

```python
import threading
import time

class WebSocketClient:
    def __init__(self, url=None):
        self.url = url
        self._ws = None
        self._heartbeat_thread = None
        self._last_activity_time = None
        self._ping_sent_time = None
        self._running = False
    
    HEARTBEAT_INTERVAL_SECONDS = 30
    PONG_TIMEOUT_SECONDS = 5
    
    def connect(self, url):
        """Connect to WebSocket server."""
        self.url = url
        self._ws = self._create_websocket_connection()
        self._last_activity_time = time.time()
        self._running = True
        
        # Start heartbeat monitor thread
        self._heartbeat_thread = threading.Thread(
            target=self._heartbeat_loop,
            daemon=True
        )
        self._heartbeat_thread.start()
    
    def _create_websocket_connection(self):
        """Create WebSocket connection (mocked in tests)."""
        import websocket
        return websocket.create_connection(self.url)
    
    def _heartbeat_loop(self):
        """Monitor heartbeat and send pings periodically."""
        while self._running:
            self._check_heartbeat()
            time.sleep(1)  # Check every second
    
    def _check_heartbeat(self):
        """Check if ping should be sent or pong timeout occurred."""
        current_time = time.time()
        
        # Check if pong timeout occurred
        if self._ping_sent_time is not None:
            time_since_ping = current_time - self._ping_sent_time
            if time_since_ping > self.PONG_TIMEOUT_SECONDS:
                self._close_on_pong_timeout()
                return
        
        # Check if time to send ping
        time_since_activity = current_time - self._last_activity_time
        if time_since_activity >= self.HEARTBEAT_INTERVAL_SECONDS:
            self._send_ping()
    
    def _send_ping(self):
        """Send ping message."""
        try:
            self._ws.send(b'ping', websocket.ABNF.OPCODE_PING)
            self._ping_sent_time = time.time()
        except Exception as e:
            self._close_connection()
    
    def _close_on_pong_timeout(self):
        """Close connection due to pong timeout."""
        self._close_connection()
    
    def _close_connection(self):
        """Close the WebSocket connection."""
        self._running = False
        try:
            self._ws.close()
        except:
            pass
    
    def send(self, message):
        """Send a message (resets heartbeat timer)."""
        self._last_activity_time = time.time()
        self._ws.send(message)
    
    def _handle_message(self, message):
        """Handle incoming message."""
        self._last_activity_time = time.time()
        
        # Filter out ping/pong
        if message in ("PING", "PONG"):
            return
        
        # Handle normal message
        if hasattr(self, 'on_message') and self.on_message:
            self.on_message(message)
```

#### Step 4: Implementer Self-Reviews

Checklist:
- [x] All tests passing
- [x] Tests describe behavior clearly
- [x] No hardcoded values (using HEARTBEAT_INTERVAL_SECONDS = 30)
- [x] Readable variable names (time_since_activity, ping_sent_time)
- [x] Error handling (try-except in send/close)
- [x] No dead code
- [x] Follows codebase conventions

Status: READY FOR SPEC REVIEW

```
Tests Written: 4
Tests Passing: 4
Code Lines: 75

Acceptance Criteria Addressed:
✓ Sends ping every 30 seconds when idle
✓ Stops sending when connection is active
✓ Closes connection if pong not received within 5 seconds
✓ Doesn't interfere with normal messages

Test Coverage:
- test_heartbeat_sends_ping_every_30_seconds
- test_heartbeat_resets_timer_on_message
- test_heartbeat_closes_connection_on_pong_timeout
- test_heartbeat_doesnt_interfere_with_normal_messages
```

#### Step 5: Spec Reviewer Reviews

```
SPEC REVIEW: APPROVED

Acceptance Criteria Verification:
✓ Criterion 1: "Sends ping every 30 seconds when idle" - VERIFIED with test_heartbeat_sends_ping_every_30_seconds
✓ Criterion 2: "Stops sending when connection is active" - VERIFIED with test_heartbeat_resets_timer_on_message
✓ Criterion 3: "Closes connection if pong not received within 5 seconds" - VERIFIED with test_heartbeat_closes_connection_on_pong_timeout
✓ Criterion 4: "Doesn't interfere with normal messages" - VERIFIED with test_heartbeat_doesnt_interfere_with_normal_messages

Test Coverage:
✓ All 4 criteria have dedicated tests
✓ Tests verify behavior, not implementation details
✓ No gaps between spec and tests

READY FOR CODE QUALITY REVIEW
```

#### Step 6: Code Quality Reviewer Reviews

```
CODE QUALITY REVIEW: APPROVED

Code Quality:
✓ Constants defined - HEARTBEAT_INTERVAL_SECONDS, PONG_TIMEOUT_SECONDS
✓ Readable names - _last_activity_time, _ping_sent_time, _heartbeat_loop
✓ Error handling - try-except in critical paths
✓ Focused functions - each has single responsibility
✓ No dead code

Test Quality:
✓ All criteria covered (4 tests)
✓ Tests verify behavior
✓ Good test names describing what's tested
✓ Edge cases covered (timeout, message interference)

Architecture:
✓ Clean separation - heartbeat logic separate from message handling
✓ Thread-safe pattern - daemon thread for monitoring
✓ Testable - uses time.time() mock for deterministic timing

APPROVED FOR PRODUCTION
```

#### Step 7: Issue Marked Done

Calls report-issue-completion:
- Issue #12 → Done
- Unblocks any issues depending on heartbeat
- Continues to next issue

---

## Example 2: Complex Task - Exponential Backoff Reconnection

### Issue Specification

**Issue #42: Implement exponential backoff reconnection for WebSocket client**

**Description**: When WebSocket connection fails, automatically reconnect with exponential backoff. First retry after 1 second, then 2s, 4s, 8s, 16s max. Give up after 5 retries.

**Acceptance Criteria**:
1. Retries with exponential backoff (1s, 2s, 4s, 8s, 16s max delay)
2. Max 5 retries before giving up
3. Cancels retries if connection succeeds
4. Logs each retry attempt with reason
5. Allows custom max retries via configuration

**Test Requirements**:
- Test successful reconnect after 2 retries
- Test exponential backoff timing
- Test max retries exceeded raises error
- Test cancellation on success
- Test custom retry count
- Test logging output

**Story Context**: Network failures are common in production. Exponential backoff prevents overwhelming a recovering server while ensuring quick recovery when possible.

### Implementation Process

#### Step 1: Implementer Clarifying Questions

"Questions about the spec:

1. When you say 'custom max retries via configuration', should this be:
   - Parameter to a function?
   - Environment variable?
   - Config file?
   - All three?

2. For logging, should we log to:
   - Standard logging module?
   - Custom logger?
   - Print statements?

3. What errors should trigger retry vs give up?
   - All exceptions?
   - Only network errors?
   - Specific list?

4. After max retries exceeded, should we:
   - Raise an exception?
   - Return False?
   - Notify via callback?"

**Response**:
"Parameter is fine for now (function arg with default). Use standard logging module. Retry on ConnectionError and TimeoutError only - let other exceptions propagate (they're bugs). Raise an exception when giving up with message indicating why."

#### Step 2: Implementer Writes Tests

```python
import pytest
import time
from unittest import mock
from websocket_client import WebSocketClient

def test_reconnect_succeeds_after_two_failures():
    """Test that reconnect succeeds after initial connection failures."""
    call_count = 0
    
    def connect_fails_twice_then_succeeds():
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise ConnectionError(f"Attempt {call_count} failed")
        return True  # Success on 3rd attempt
    
    client = WebSocketClient()
    result = client.reconnect(
        connect_func=connect_fails_twice_then_succeeds,
        max_retries=5
    )
    
    assert result is True
    assert call_count == 3

def test_reconnect_backoff_timing_is_exponential():
    """Test that backoff delays are exponential: 1s, 2s, 4s, 8s, 16s."""
    with mock.patch('time.sleep') as mock_sleep:
        def always_fails():
            raise ConnectionError("Always fails")
        
        client = WebSocketClient()
        try:
            client.reconnect(connect_func=always_fails, max_retries=5)
        except ConnectionError:
            pass
        
        # Extract delay arguments from sleep calls
        delays = [call[0][0] for call in mock_sleep.call_args_list]
        expected = [1, 2, 4, 8, 16]
        
        assert delays == expected

def test_reconnect_gives_up_after_max_retries():
    """Test that reconnect gives up after max retries exceeded."""
    def always_fails():
        raise ConnectionError("Failed")
    
    client = WebSocketClient()
    
    with pytest.raises(ConnectionError, match="Max retries exceeded"):
        client.reconnect(connect_func=always_fails, max_retries=2)

def test_reconnect_cancels_on_success():
    """Test that reconnect stops retrying once connection succeeds."""
    call_count = 0
    
    def fails_once_then_succeeds():
        nonlocal call_count
        call_count += 1
        if call_count < 2:
            raise ConnectionError("First attempt failed")
        return True
    
    client = WebSocketClient()
    result = client.reconnect(
        connect_func=fails_once_then_succeeds,
        max_retries=5
    )
    
    assert result is True
    # Should only be called twice (failed once, succeeded once)
    # Not all 5 retries
    assert call_count == 2

def test_reconnect_respects_custom_max_retries():
    """Test that custom max_retries parameter is respected."""
    call_count = 0
    
    def always_fails():
        nonlocal call_count
        call_count += 1
        raise ConnectionError("Failed")
    
    client = WebSocketClient()
    
    with pytest.raises(ConnectionError):
        client.reconnect(connect_func=always_fails, max_retries=2)
    
    # Initial attempt + 2 retries = 3 total calls
    assert call_count == 3

def test_reconnect_logs_each_attempt():
    """Test that each reconnection attempt is logged."""
    with mock.patch('logging.info') as mock_log:
        def always_fails():
            raise ConnectionError("Network error")
        
        client = WebSocketClient()
        
        try:
            client.reconnect(connect_func=always_fails, max_retries=2)
        except ConnectionError:
            pass
        
        # Should log initial attempt and each retry
        assert mock_log.call_count >= 3  # Initial + 2 retries
        
        # Verify log messages mention retry attempts
        log_calls = [str(call) for call in mock_log.call_args_list]
        assert any('retry' in log.lower() for log in log_calls)
        assert any('Network error' in log for log in log_calls)
```

#### Step 3: Implementer Implements

```python
import logging
import time
from typing import Callable, Optional

logger = logging.getLogger(__name__)

class WebSocketClient:
    DEFAULT_MAX_RETRIES = 5
    MAX_BACKOFF_DELAY_SECONDS = 16
    BACKOFF_MULTIPLIER = 2
    INITIAL_BACKOFF_SECONDS = 1
    
    def reconnect(
        self,
        connect_func: Callable,
        max_retries: Optional[int] = None
    ) -> bool:
        """Reconnect with exponential backoff.
        
        Retries with exponential backoff: 1s, 2s, 4s, 8s, 16s max.
        
        Args:
            connect_func: Callable that attempts connection. Raises on failure.
            max_retries: Maximum retries. Defaults to DEFAULT_MAX_RETRIES.
        
        Returns:
            True if connection successful.
        
        Raises:
            ConnectionError: If all retries exhausted.
            TimeoutError: If pong timeout (from heartbeat).
            Other exceptions: Propagated as-is (indicate bugs).
        """
        if max_retries is None:
            max_retries = self.DEFAULT_MAX_RETRIES
        
        last_error = None
        
        # Initial attempt
        attempt = 0
        try:
            logger.info(f"Attempting connection (initial)")
            result = connect_func()
            logger.info(f"Connection successful")
            return result
        except (ConnectionError, TimeoutError) as e:
            last_error = e
            logger.warning(f"Initial connection failed: {e}")
        
        # Retries with exponential backoff
        for attempt in range(1, max_retries + 1):
            # Calculate backoff delay
            backoff_exponent = attempt - 1
            delay = min(
                self.INITIAL_BACKOFF_SECONDS * (self.BACKOFF_MULTIPLIER ** backoff_exponent),
                self.MAX_BACKOFF_DELAY_SECONDS
            )
            
            logger.info(
                f"Retry {attempt}/{max_retries}: "
                f"waiting {delay}s before reconnecting. "
                f"Last error: {last_error}"
            )
            
            time.sleep(delay)
            
            try:
                logger.info(f"Attempting connection (retry {attempt})")
                result = connect_func()
                logger.info(f"Connection successful after {attempt} retries")
                return result
            except (ConnectionError, TimeoutError) as e:
                last_error = e
                logger.warning(
                    f"Retry {attempt} failed after {delay}s: {e}"
                )
        
        # All retries exhausted
        logger.error(
            f"Max retries ({max_retries}) exceeded. "
            f"Last error: {last_error}"
        )
        raise ConnectionError(
            f"Max retries ({max_retries}) exceeded. "
            f"Last error: {last_error}"
        )
```

#### Step 4: Implementer Self-Reviews

Checklist:
- [x] All tests passing (6/6)
- [x] Tests describe behavior clearly
- [x] Constants defined (INITIAL_BACKOFF_SECONDS, BACKOFF_MULTIPLIER, MAX_BACKOFF_DELAY_SECONDS)
- [x] No hardcoded values
- [x] Readable variable names
- [x] Error handling specific to expected errors
- [x] Logging throughout
- [x] No dead code
- [x] Docstring explains behavior

Status: READY FOR SPEC REVIEW

```
Tests Written: 6
Tests Passing: 6
Code Lines: 85

Acceptance Criteria Addressed:
✓ Exponential backoff (1s, 2s, 4s, 8s, 16s max)
✓ Max 5 retries
✓ Cancels on success
✓ Logs each attempt
✓ Custom retry count parameter

Test Coverage:
- test_reconnect_succeeds_after_two_failures
- test_reconnect_backoff_timing_is_exponential
- test_reconnect_gives_up_after_max_retries
- test_reconnect_cancels_on_success
- test_reconnect_respects_custom_max_retries
- test_reconnect_logs_each_attempt
```

#### Step 5: Spec Reviewer Reviews - FIRST SUBMISSION REJECTED

```
SPEC REVIEW: ISSUES FOUND

1. CRITICAL - Test Gap: "Retries only on network errors, not others"
   Current: Code catches ConnectionError and TimeoutError ✓
   But no test verifies this!
   Missing: Test that other exceptions propagate (not retried)
   
   Example: If code raises ValueError (bug), it should fail immediately,
   not retry 5 times.
   
   Fix: Add test_reconnect_does_not_retry_on_programming_errors()

2. Test Quality: test_reconnect_backoff_timing_is_exponential
   Issue: Test assumes 5 retries to get all delays, but max_retries=5 
          means 5 retries AFTER initial, so 6 total attempts.
   Current test expects: [1, 2, 4, 8, 16] (5 delays)
   Actually happens: [1, 2, 4, 8, 16, ?] (6 delays, exceeds MAX_BACKOFF_DELAY_SECONDS)
   
   Fix: Clarify test - does it test up to max_retries or actual sequence?
        I recommend: test 2 retries to verify timing works, 
        separate test for 5 retries touching MAX_BACKOFF_DELAY_SECONDS

NEXT STEPS:
1. Add test for non-network exceptions (shouldn't retry)
2. Fix backoff timing test for clarity
3. Verify all tests pass
4. Resubmit

Questions? Ask before fixing.
```

#### Step 6: Implementer Fixes Issues

Added test for programming errors:
```python
def test_reconnect_does_not_retry_on_programming_errors():
    """Test that programming errors are not retried, fail immediately."""
    def raises_value_error():
        raise ValueError("Programming error - invalid parameter")
    
    client = WebSocketClient()
    
    # Should raise immediately, not retry
    with pytest.raises(ValueError, match="Programming error"):
        client.reconnect(connect_func=raises_value_error, max_retries=5)
```

Fixed timing test:
```python
def test_reconnect_backoff_timing_with_multiple_retries():
    """Test exponential backoff timing with 2 retries: 1s, 2s."""
    with mock.patch('time.sleep') as mock_sleep:
        def fails_three_times():
            raise ConnectionError("Failed")
        
        client = WebSocketClient()
        try:
            client.reconnect(connect_func=fails_three_times, max_retries=2)
        except ConnectionError:
            pass
        
        # Initial attempt (no delay) + 2 retries = 2 sleep calls
        delays = [call[0][0] for call in mock_sleep.call_args_list]
        assert delays == [1, 2]
```

Status: READY FOR SPEC REVIEW (RESUBMISSION)

#### Step 7: Spec Reviewer Reviews - APPROVED

```
SPEC REVIEW: APPROVED

Acceptance Criteria Verification:
✓ Criterion 1: "Exponential backoff" - VERIFIED with test_reconnect_backoff_timing_with_multiple_retries
✓ Criterion 2: "Max 5 retries" - VERIFIED with test_reconnect_gives_up_after_max_retries
✓ Criterion 3: "Cancels on success" - VERIFIED with test_reconnect_cancels_on_success
✓ Criterion 4: "Logs each attempt" - VERIFIED with test_reconnect_logs_each_attempt
✓ Criterion 5: "Custom retry count" - VERIFIED with test_reconnect_respects_custom_max_retries

Test Coverage:
✓ All 5 criteria tested
✓ Edge case: Programming errors don't retry ✓
✓ No gaps between spec and tests

READY FOR CODE QUALITY REVIEW
```

#### Step 8: Code Quality Reviewer Reviews

```
CODE QUALITY REVIEW: APPROVED

Code Quality:
✓ Constants defined - INITIAL_BACKOFF_SECONDS, BACKOFF_MULTIPLIER, etc
✓ No magic numbers - all values are named constants
✓ Readable names - backoff_exponent, attempt, delay
✓ Error handling comprehensive - catches specific exceptions, propagates others
✓ Helpful error messages - explains why it failed
✓ Logging at appropriate levels (info, warning, error)
✓ No dead code

Test Quality:
✓ 7 tests total
✓ Tests verify behavior, not implementation
✓ Good test names - describe exactly what's tested
✓ Edge cases covered:
  - Success after failures
  - Cancellation on success
  - Custom retry count
  - Programming errors don't retry
  - Proper backoff timing

Architecture:
✓ Single responsibility - reconnect function focuses on retry logic
✓ Testable - uses time.sleep mock, injected connect_func
✓ Extensible - easy to adjust constants if needed
✓ Thread-safe - stateless function

Performance:
✓ Minimal delays - only waits between retries
✓ Reasonable backoff - grows exponentially but caps at 16s
✓ No busy loops

APPROVED FOR PRODUCTION

This is clean, well-tested, production-ready code that handles the complexity
of exponential backoff reconnection clearly and thoroughly.
```

#### Step 9: Issue Marked Done

Calls report-issue-completion:
- Issue #42 → Done
- Unblocks any issues depending on exponential backoff
- Continues to next issue

---

## Key Learnings from Examples

### Example 1 (Simple)
- TDD is straightforward for simple features
- One test per criterion is often sufficient
- Self-review usually passes without issues
- Single reviewer round is common

### Example 2 (Complex)
- Complex features may need review iterations
- Rejections are specific and actionable
- Implementer can ask clarifying questions
- Multiple tests per criterion is common
- Edge cases (error types) matter
- Both reviewer rounds are important

### Both Examples Show
- Tests written BEFORE implementation
- Clear acceptance criteria → clear tests
- Spec reviewer focuses on compliance
- Quality reviewer focuses on production readiness
- No pausing between reviews - linear workflow
- Rejection is specific and solvable

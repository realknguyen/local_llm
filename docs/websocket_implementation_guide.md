# WebSocket Implementation Guide: Handling 500+ Subscriptions

This document outlines the best practices and implementation approach for creating a Python WebSocket server that can efficiently handle more than 500 concurrent subscriptions while running in the background and waiting for WebSocket data.

## Overview

For applications that need to handle a large number of WebSocket connections (such as real-time cryptocurrency market data feeds), it's essential to use asynchronous programming with proper connection management. The key is to use Python's `asyncio` with the `websockets` library to efficiently manage concurrent connections without blocking.

## Recommended Architecture

### Core Components

1. **Async WebSocket Server**: Using `websockets` library with `asyncio`
2. **Connection Manager**: Tracks active connections and subscriptions
3. **Background Task Processor**: Handles data processing without blocking the WebSocket loop
4. **Message Distribution**: Efficiently routes messages to subscribed clients
5. **Resource Management**: Proper cleanup and connection limits

## Implementation Approach

### 1. Using websockets library with asyncio

```python
import asyncio
import websockets
import json
from collections import defaultdict
from typing import Set, Dict
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MarketDataWebSocketServer:
    def __init__(self, host="localhost", port=8765):
        self.host = host
        self.port = port
        self.clients: Set[websockets.WebSocketServerProtocol] = set()
        self.subscriptions: Dict[str, Set[websockets.WebSocketServerProtocol]] = defaultdict(set)
        self.message_queue = asyncio.Queue()
        self.background_tasks = set()
        
    async def register_client(self, websocket):
        """Register a new client connection"""
        self.clients.add(websocket)
        logger.info(f"New client connected. Total clients: {len(self.clients)}")
        
        # Start background task for message processing
        task = asyncio.create_task(self.handle_client_messages(websocket))
        self.background_tasks.add(task)
        
        try:
            # Send initial subscription info or welcome message
            await websocket.send(json.dumps({
                "type": "welcome",
                "message": "Connected to market data feed",
                "client_count": len(self.clients)
            }))
        except Exception as e:
            logger.error(f"Error sending welcome message: {e}")
            
    async def unregister_client(self, websocket):
        """Unregister a client connection"""
        self.clients.discard(websocket)
        
        # Remove from all subscriptions
        for exchange, subs in self.subscriptions.items():
            subs.discard(websocket)
            
        # Cancel background task
        for task in self.background_tasks:
            if not task.done():
                task.cancel()
        self.background_tasks.clear()
        
        logger.info(f"Client disconnected. Total clients: {len(self.clients)}")
        
    async def handle_client_messages(self, websocket):
        """Handle messages from a specific client"""
        try:
            async for message in websocket:
                try:
                    data = json.loads(message)
                    await self.process_client_message(websocket, data)
                except json.JSONDecodeError:
                    logger.error("Invalid JSON received")
                    await websocket.send(json.dumps({
                        "type": "error",
                        "message": "Invalid JSON format"
                    }))
        except websockets.exceptions.ConnectionClosed:
            logger.info("Client connection closed")
        except Exception as e:
            logger.error(f"Error handling client messages: {e}")
        finally:
            await self.unregister_client(websocket)
            
    async def process_client_message(self, websocket, data):
        """Process messages from clients"""
        message_type = data.get("type")
        
        if message_type == "subscribe":
            exchange = data.get("exchange")
            symbol = data.get("symbol")
            
            if exchange and symbol:
                key = f"{exchange}:{symbol}"
                self.subscriptions[key].add(websocket)
                logger.info(f"Client subscribed to {key}")
                
                await websocket.send(json.dumps({
                    "type": "subscribed",
                    "exchange": exchange,
                    "symbol": symbol
                }))
                
        elif message_type == "unsubscribe":
            exchange = data.get("exchange")
            symbol = data.get("symbol")
            
            if exchange and symbol:
                key = f"{exchange}:{symbol}"
                self.subscriptions[key].discard(websocket)
                logger.info(f"Client unsubscribed from {key}")
                
                await websocket.send(json.dumps({
                    "type": "unsubscribed",
                    "exchange": exchange,
                    "symbol": symbol
                }))
                
    async def broadcast_message(self, message: dict, exchange: str = None, symbol: str = None):
        """Broadcast message to relevant subscribers"""
        if exchange and symbol:
            # Send to specific subscription
            key = f"{exchange}:{symbol}"
            subscribers = self.subscriptions.get(key, set())
            await self._send_to_subscribers(subscribers, message)
        else:
            # Send to all clients
            await self._send_to_subscribers(self.clients, message)
            
    async def _send_to_subscribers(self, subscribers: Set[websockets.WebSocketServerProtocol], message: dict):
        """Send message to a set of subscribers"""
        if not subscribers:
            return
            
        # Create tasks for sending messages
        tasks = []
        for subscriber in list(subscribers):  # Create a copy to avoid modification during iteration
            try:
                task = asyncio.create_task(subscriber.send(json.dumps(message)))
                tasks.append(task)
            except Exception as e:
                logger.error(f"Error sending to subscriber: {e}")
                # Remove broken connections
                subscribers.discard(subscriber)
                
        # Wait for all sends to complete
        await asyncio.gather(*tasks, return_exceptions=True)
        
    async def start_server(self):
        """Start the WebSocket server"""
        server = await websockets.serve(
            self.handle_connection,
            self.host,
            self.port,
            ping_interval=20,  # Send ping every 20 seconds
            ping_timeout=10,    # Wait 10 seconds for pong
            close_timeout=5,      # Wait 5 seconds for close handshake
            max_size=10 * 1024 * 1024  # 10MB max message size
        )
        
        logger.info(f"WebSocket server started on {self.host}:{self.port}")
        await server.wait_closed()
        
    async def handle_connection(self, websocket, path):
        """Handle new WebSocket connections"""
        await self.register_client(websocket)
        
        try:
            # Keep connection alive
            async for message in websocket:
                # This should not be reached due to handle_client_messages
                pass
        except websockets.exceptions.ConnectionClosed:
            logger.info("Connection closed")
        except Exception as e:
            logger.error(f"Connection error: {e}")
        finally:
            await self.unregister_client(websocket)

# Example usage for background data processing
async def background_data_processor(server: MarketDataWebSocketServer):
    """Background task that simulates receiving market data"""
    while True:
        try:
            # Simulate receiving market data from exchanges
            # In a real implementation, this would connect to actual exchange APIs
            await asyncio.sleep(1)  # Simulate data processing delay
            
            # Example market data
            market_data = {
                "type": "market_update",
                "exchange": "binance",
                "symbol": "BTCUSDT",
                "price": 67890.50,
                "volume": 1234.56,
                "timestamp": asyncio.get_event_loop().time()
            }
            
            # Broadcast to relevant subscribers
            await server.broadcast_message(market_data, "binance", "BTCUSDT")
            
        except Exception as e:
            logger.error(f"Background data processing error: {e}")
            await asyncio.sleep(5)  # Wait before retrying

async def main():
    """Main function to start server and background tasks"""
    server = MarketDataWebSocketServer("localhost", 8765)
    
    # Start background data processing task
    data_task = asyncio.create_task(background_data_processor(server))
    
    # Start WebSocket server
    try:
        await server.start_server()
    except KeyboardInterrupt:
        logger.info("Shutting down server...")
        data_task.cancel()
        try:
            await data_task
        except asyncio.CancelledError:
            logger.info("Data processor cancelled")

if __name__ == "__main__":
    asyncio.run(main())
```

### 2. Key Configuration Options

```python
# Server configuration for high connection counts
server = await websockets.serve(
    handler,
    host,
    port,
    ping_interval=20,        # Send ping every 20 seconds
    ping_timeout=10,         # Wait 10 seconds for pong
    close_timeout=5,         # Wait 5 seconds for close handshake
    max_size=10 * 1024 * 1024,  # 10MB max message size
    # For very high connection counts, consider:
    # read_limit=65536,      # 64KB read limit
    # write_limit=65536,     # 64KB write limit
    # compression=None,       # Disable compression if not needed
)
```

### 3. Performance Optimization Tips

1. **Connection Limits**: Set appropriate limits to prevent resource exhaustion
2. **Memory Management**: Use weak references for large data structures
3. **Message Batching**: Batch messages to reduce network overhead
4. **Connection Heartbeats**: Implement ping/pong for connection health
5. **Graceful Shutdown**: Properly close connections during shutdown

### 4. Running in Background

To run this as a background service:

```python
import asyncio
import sys
import os

def run_server():
    """Run the server in background"""
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("Server stopped by user")
        sys.exit(0)
    except Exception as e:
        print(f"Server error: {e}")
        sys.exit(1)

if __name__ == "__main__":
    # Run in background
    import multiprocessing
    if len(sys.argv) > 1 and sys.argv[1] == "--background":
        # Fork background process
        pid = os.fork()
        if pid == 0:
            # Child process
            run_server()
        else:
            # Parent process - save PID
            with open("websocket_server.pid", "w") as f:
                f.write(str(pid))
            print(f"Server started with PID {pid}")
    else:
        run_server()
```

## Best Practices for 500+ Connections

1. **Use Async/Await**: Never block the event loop
2. **Connection Pooling**: Reuse connections where possible
3. **Memory Efficiency**: Use generators and streaming for large data
4. **Error Handling**: Always handle connection drops gracefully
5. **Resource Limits**: Set appropriate limits for connections, message sizes, and timeouts
6. **Monitoring**: Add logging and metrics for connection health
7. **Graceful Shutdown**: Implement proper cleanup on shutdown
8. **Load Testing**: Test with simulated connections before production

## Required Dependencies

```txt
websockets>=10.0
asyncio
```

## Deployment Considerations

1. **System Limits**: Increase file descriptor limits:
   ```bash
   ulimit -n 65536
   ```

2. **Server Configuration**: For production, consider using:
   - Uvicorn with FastAPI for better performance
   - Load balancer if running multiple instances
   - Proper monitoring and alerting

3. **Security**: 
   - Implement authentication/authorization
   - Use TLS/SSL for secure connections
   - Rate limiting to prevent abuse

This approach allows handling of 500+ concurrent WebSocket connections efficiently while maintaining low latency and high throughput for real-time market data distribution.
This document outlines the best practices and implementation approach for creating a Python WebSocket server that can efficiently handle more than 500 concurrent subscriptions while running in the background and waiting for WebSocket data.

## Overview

For applications that need to handle a large number of WebSocket connections (such as real-time cryptocurrency market data feeds), it's essential to use asynchronous programming with proper connection management. The key is to use Python's `asyncio` with the `websockets` library to efficiently manage concurrent connections without blocking.

## Recommended Architecture

### Core Components

1. **Async WebSocket Server**: Using `websockets` library with `asyncio`
2. **Connection Manager**: Tracks active connections and subscriptions
3. **Background Task Processor**: Handles data processing without blocking the WebSocket loop
4. **Message Distribution**: Efficiently routes messages to subscribed clients
5. **Resource Management**: Proper cleanup and connection limits

## Implementation Approach

### 1. Using websockets library with asyncio

```python
import asyncio
import websockets
import json
from collections import defaultdict
from typing import Set, Dict
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MarketDataWebSocketServer:
    def __init__(self, host="localhost", port=8765):
        self.host = host
        self.port = port
        self.clients: Set[websockets.WebSocketServerProtocol] = set()
        self.subscriptions: Dict[str, Set[websockets.WebSocketServerProtocol]] = defaultdict(set)
        self.message_queue = asyncio.Queue()
        self.background_tasks = set()
        
    async def register_client(self, websocket):
        """Register a new client connection"""
        self.clients.add(websocket)
        logger.info(f"New client connected. Total clients: {len(self.clients)}")
        
        # Start background task for message processing
        task = asyncio.create_task(self.handle_client_messages(websocket))
        self.background_tasks.add(task)
        
        try:
            # Send initial subscription info or welcome message
            await websocket.send(json.dumps({
                "type": "welcome",
                "message": "Connected to market data feed",
                "client_count": len(self.clients)
            }))
        except Exception as e:
            logger.error(f"Error sending welcome message: {e}")
            
    async def unregister_client(self, websocket):
        """Unregister a client connection"""
        self.clients.discard(websocket)
        
        # Remove from all subscriptions
        for exchange, subs in self.subscriptions.items():
            subs.discard(websocket)
            
        # Cancel background task
        for task in self.background_tasks:
            if not task.done():
                task.cancel()
        self.background_tasks.clear()
        
        logger.info(f"Client disconnected. Total clients: {len(self.clients)}")
        
    async def handle_client_messages(self, websocket):
        """Handle messages from a specific client"""
        try:
            async for message in websocket:
                try:
                    data = json.loads(message)
                    await self.process_client_message(websocket, data)
                except json.JSONDecodeError:
                    logger.error("Invalid JSON received")
                    await websocket.send(json.dumps({
                        "type": "error",
                        "message": "Invalid JSON format"
                    }))
        except websockets.exceptions.ConnectionClosed:
            logger.info("Client connection closed")
        except Exception as e:
            logger.error(f"Error handling client messages: {e}")
        finally:
            await self.unregister_client(websocket)
            
    async def process_client_message(self, websocket, data):
        """Process messages from clients"""
        message_type = data.get("type")
        
        if message_type == "subscribe":
            exchange = data.get("exchange")
            symbol = data.get("symbol")
            
            if exchange and symbol:
                key = f"{exchange}:{symbol}"
                self.subscriptions[key].add(websocket)
                logger.info(f"Client subscribed to {key}")
                
                await websocket.send(json.dumps({
                    "type": "subscribed",
                    "exchange": exchange,
                    "symbol": symbol
                }))
                
        elif message_type == "unsubscribe":
            exchange = data.get("exchange")
            symbol = data.get("symbol")
            
            if exchange and symbol:
                key = f"{exchange}:{symbol}"
                self.subscriptions[key].discard(websocket)
                logger.info(f"Client unsubscribed from {key}")
                
                await websocket.send(json.dumps({
                    "type": "unsubscribed",
                    "exchange": exchange,
                    "symbol": symbol
                }))
                
    async def broadcast_message(self, message: dict, exchange: str = None, symbol: str = None):
        """Broadcast message to relevant subscribers"""
        if exchange and symbol:
            # Send to specific subscription
            key = f"{exchange}:{symbol}"
            subscribers = self.subscriptions.get(key, set())
            await self._send_to_subscribers(subscribers, message)
        else:
            # Send to all clients
            await self._send_to_subscribers(self.clients, message)
            
    async def _send_to_subscribers(self, subscribers: Set[websockets.WebSocketServerProtocol], message: dict):
        """Send message to a set of subscribers"""
        if not subscribers:
            return
            
        # Create tasks for sending messages
        tasks = []
        for subscriber in list(subscribers):  # Create a copy to avoid modification during iteration
            try:
                task = asyncio.create_task(subscriber.send(json.dumps(message)))
                tasks.append(task)
            except Exception as e:
                logger.error(f"Error sending to subscriber: {e}")
                # Remove broken connections
                subscribers.discard(subscriber)
                
        # Wait for all sends to complete
        await asyncio.gather(*tasks, return_exceptions=True)
        
    async def start_server(self):
        """Start the WebSocket server"""
        server = await websockets.serve(
            self.handle_connection,
            self.host,
            self.port,
            ping_interval=20,  # Send ping every 20 seconds
            ping_timeout=10,    # Wait 10 seconds for pong
            close_timeout=5,      # Wait 5 seconds for close handshake
            max_size=10 * 1024 * 1024  # 10MB max message size
        )
        
        logger.info(f"WebSocket server started on {self.host}:{self.port}")
        await server.wait_closed()
        
    async def handle_connection(self, websocket, path):
        """Handle new WebSocket connections"""
        await self.register_client(websocket)
        
        try:
            # Keep connection alive
            async for message in websocket:
                # This should not be reached due to handle_client_messages
                pass
        except websockets.exceptions.ConnectionClosed:
            logger.info("Connection closed")
        except Exception as e:
            logger.error(f"Connection error: {e}")
        finally:
            await self.unregister_client(websocket)

# Example usage for background data processing
async def background_data_processor(server: MarketDataWebSocketServer):
    """Background task that simulates receiving market data"""
    while True:
        try:
            # Simulate receiving market data from exchanges
            # In a real implementation, this would connect to actual exchange APIs
            await asyncio.sleep(1)  # Simulate data processing delay
            
            # Example market data
            market_data = {
                "type": "market_update",
                "exchange": "binance",
                "symbol": "BTCUSDT",
                "price": 67890.50,
                "volume": 1234.56,
                "timestamp": asyncio.get_event_loop().time()
            }
            
            # Broadcast to relevant subscribers
            await server.broadcast_message(market_data, "binance", "BTCUSDT")
            
        except Exception as e:
            logger.error(f"Background data processing error: {e}")
            await asyncio.sleep(5)  # Wait before retrying

async def main():
    """Main function to start server and background tasks"""
    server = MarketDataWebSocketServer("localhost", 8765)
    
    # Start background data processing task
    data_task = asyncio.create_task(background_data_processor(server))
    
    # Start WebSocket server
    try:
        await server.start_server()
    except KeyboardInterrupt:
        logger.info("Shutting down server...")
        data_task.cancel()
        try:
            await data_task
        except asyncio.CancelledError:
            logger.info("Data processor cancelled")

if __name__ == "__main__":
    asyncio.run(main())
```

### 2. Key Configuration Options

```python
# Server configuration for high connection counts
server = await websockets.serve(
    handler,
    host,
    port,
    ping_interval=20,        # Send ping every 20 seconds
    ping_timeout=10,         # Wait 10 seconds for pong
    close_timeout=5,         # Wait 5 seconds for close handshake
    max_size=10 * 1024 * 1024,  # 10MB max message size
    # For very high connection counts, consider:
    # read_limit=65536,      # 64KB read limit
    # write_limit=65536,     # 64KB write limit
    # compression=None,       # Disable compression if not needed
)
```

### 3. Performance Optimization Tips

1. **Connection Limits**: Set appropriate limits to prevent resource exhaustion
2. **Memory Management**: Use weak references for large data structures
3. **Message Batching**: Batch messages to reduce network overhead
4. **Connection Heartbeats**: Implement ping/pong for connection health
5. **Graceful Shutdown**: Properly close connections during shutdown

### 4. Running in Background

To run this as a background service:

```python
import asyncio
import sys
import os

def run_server():
    """Run the server in background"""
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("Server stopped by user")
        sys.exit(0)
    except Exception as e:
        print(f"Server error: {e}")
        sys.exit(1)

if __name__ == "__main__":
    # Run in background
    import multiprocessing
    if len(sys.argv) > 1 and sys.argv[1] == "--background":
        # Fork background process
        pid = os.fork()
        if pid == 0:
            # Child process
            run_server()
        else:
            # Parent process - save PID
            with open("websocket_server.pid", "w") as f:
                f.write(str(pid))
            print(f"Server started with PID {pid}")
    else:
        run_server()
```

## Best Practices for 500+ Connections

1. **Use Async/Await**: Never block the event loop
2. **Connection Pooling**: Reuse connections where possible
3. **Memory Efficiency**: Use generators and streaming for large data
4. **Error Handling**: Always handle connection drops gracefully
5. **Resource Limits**: Set appropriate limits for connections, message sizes, and timeouts
6. **Monitoring**: Add logging and metrics for connection health
7. **Graceful Shutdown**: Implement proper cleanup on shutdown
8. **Load Testing**: Test with simulated connections before production

## Required Dependencies

```txt
websockets>=10.0
asyncio
```

## Deployment Considerations

1. **System Limits**: Increase file descriptor limits:
   ```bash
   ulimit -n 65536
   ```

2. **Server Configuration**: For production, consider using:
   - Uvicorn with FastAPI for better performance
   - Load balancer if running multiple instances
   - Proper monitoring and alerting

3. **Security**: 
   - Implement authentication/authorization
   - Use TLS/SSL for secure connections
   - Rate limiting to prevent abuse

This approach allows handling of 500+ concurrent WebSocket connections efficiently while maintaining low latency and high throughput for real-time market data distribution.
This approach allows handling of 500+ concurrent WebSocket connections efficiently while maintaining low latency and high throughput for real-time market data distribution.

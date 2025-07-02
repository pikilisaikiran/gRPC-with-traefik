# gRPC Client Examples

## Overview

This document provides examples of how to interact with your deployed gRPC service using various clients and programming languages.

## Service Information

**Service Endpoint**: `grpc-service.example.com:443`  
**Available Services**:
- `grpc.reflection.v1alpha.ServerReflection`
- `helloworld.helloworld`

**Available Methods**:
- `helloworld.helloworld/GetServerResponse`

## Command Line Examples (grpcurl)

### Basic Service Discovery
```bash
# List all available services
grpcurl grpc-service.example.com:443 list

# Describe a specific service
grpcurl grpc-service.example.com:443 describe helloworld.helloworld

# Describe message types
grpcurl grpc-service.example.com:443 describe helloworld.Message
grpcurl grpc-service.example.com:443 describe helloworld.MessageResponse
```

### Method Invocation Examples
```bash
# Basic method call
grpcurl -d '{"message": "Hello World"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse

# Pretty-printed JSON output
grpcurl -d '{"message": "Hello World"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse | jq .

# With custom headers
grpcurl -H "Custom-Header: MyValue" \
  -d '{"message": "Hello with headers"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse

# Verbose output (shows headers and timing)
grpcurl -v -d '{"message": "Verbose test"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse
```

### Testing Different Message Types
```bash
# Short message
grpcurl -d '{"message": "Hi"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse

# Long message
grpcurl -d '{"message": "This is a much longer message to test how the gRPC service handles larger payloads and whether there are any size limitations or performance impacts."}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse

# Empty message
grpcurl -d '{"message": ""}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse

# Unicode message
grpcurl -d '{"message": "Hello 世界 🌍 Здравствуй мир"}' \
  grpc-service.example.com:443 \
  helloworld.helloworld/GetServerResponse
```

## Python Client Examples

### Basic Python Client
```python
#!/usr/bin/env python3
# grpc_client.py

import grpc
import helloworld_pb2
import helloworld_pb2_grpc
import ssl

def run_client():
    # Create SSL credentials
    credentials = grpc.ssl_channel_credentials()
    
    # Create channel with SSL
    with grpc.secure_channel('grpc-service.example.com:443', credentials) as channel:
        # Create stub
        stub = helloworld_pb2_grpc.helloworldStub(channel)
        
        # Create request
        request = helloworld_pb2.Message(message="Hello from Python client")
        
        # Make the call
        try:
            response = stub.GetServerResponse(request)
            print(f"Response: {response.message}")
            print(f"Received: {response.received}")
        except grpc.RpcError as e:
            print(f"gRPC error: {e.code()}, {e.details()}")

if __name__ == '__main__':
    run_client()
```

### Advanced Python Client with Error Handling
```python
#!/usr/bin/env python3
# advanced_grpc_client.py

import grpc
import helloworld_pb2
import helloworld_pb2_grpc
import ssl
import time
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class GrpcClient:
    def __init__(self, server_address):
        self.server_address = server_address
        self.credentials = grpc.ssl_channel_credentials()
        
    def create_channel(self):
        """Create a secure gRPC channel"""
        options = [
            ('grpc.keepalive_time_ms', 30000),
            ('grpc.keepalive_timeout_ms', 5000),
            ('grpc.keepalive_permit_without_calls', True),
            ('grpc.http2.max_pings_without_data', 0),
            ('grpc.http2.min_time_between_pings_ms', 10000),
            ('grpc.http2.min_ping_interval_without_data_ms', 300000)
        ]
        
        return grpc.secure_channel(
            self.server_address, 
            self.credentials,
            options=options
        )
    
    def call_service(self, message, timeout=10):
        """Make a gRPC call with error handling"""
        try:
            with self.create_channel() as channel:
                stub = helloworld_pb2_grpc.helloworldStub(channel)
                request = helloworld_pb2.Message(message=message)
                
                logger.info(f"Sending request: {message}")
                start_time = time.time()
                
                response = stub.GetServerResponse(
                    request, 
                    timeout=timeout,
                    metadata=[('client-id', 'python-advanced-client')]
                )
                
                end_time = time.time()
                logger.info(f"Response time: {(end_time - start_time)*1000:.2f}ms")
                
                return response
                
        except grpc.RpcError as e:
            logger.error(f"gRPC error: {e.code()} - {e.details()}")
            raise
        except Exception as e:
            logger.error(f"Unexpected error: {e}")
            raise
    
    def health_check(self):
        """Simple health check"""
        try:
            response = self.call_service("Health check", timeout=5)
            return response.received
        except:
            return False

def main():
    client = GrpcClient('grpc-service.example.com:443')
    
    # Health check
    if client.health_check():
        logger.info("✓ Service is healthy")
    else:
        logger.error("✗ Service health check failed")
        return
    
    # Test various messages
    test_messages = [
        "Hello World",
        "Testing with special chars: !@#$%^&*()",
        "Unicode test: 你好世界",
        "Long message: " + "x" * 1000,
        ""
    ]
    
    for msg in test_messages:
        try:
            response = client.call_service(msg)
            logger.info(f"Success: {response.message[:100]}...")
        except Exception as e:
            logger.error(f"Failed for message '{msg[:20]}...': {e}")

if __name__ == '__main__':
    main()
```

### Async Python Client
```python
#!/usr/bin/env python3
# async_grpc_client.py

import asyncio
import grpc
import helloworld_pb2
import helloworld_pb2_grpc
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AsyncGrpcClient:
    def __init__(self, server_address):
        self.server_address = server_address
        self.credentials = grpc.ssl_channel_credentials()
    
    async def call_service_async(self, message):
        """Make an async gRPC call"""
        async with grpc.aio.secure_channel(
            self.server_address, 
            self.credentials
        ) as channel:
            stub = helloworld_pb2_grpc.helloworldStub(channel)
            request = helloworld_pb2.Message(message=message)
            
            try:
                response = await stub.GetServerResponse(request)
                return response
            except grpc.RpcError as e:
                logger.error(f"gRPC error: {e.code()} - {e.details()}")
                raise

async def concurrent_calls():
    """Make multiple concurrent gRPC calls"""
    client = AsyncGrpcClient('grpc-service.example.com:443')
    
    # Create multiple concurrent calls
    tasks = []
    for i in range(10):
        task = client.call_service_async(f"Concurrent call {i}")
        tasks.append(task)
    
    # Wait for all calls to complete
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # Process results
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            logger.error(f"Call {i} failed: {result}")
        else:
            logger.info(f"Call {i} success: {result.message[:50]}...")

if __name__ == '__main__':
    asyncio.run(concurrent_calls())
```

## Go Client Examples

### Basic Go Client
```go
// main.go
package main

import (
    "context"
    "crypto/tls"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    pb "path/to/your/proto/helloworld"
)

func main() {
    // Create TLS credentials
    config := &tls.Config{
        ServerName: "grpc-service.example.com",
    }
    creds := credentials.NewTLS(config)

    // Connect to the server
    conn, err := grpc.Dial("grpc-service.example.com:443", grpc.WithTransportCredentials(creds))
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    // Create client
    client := pb.NewHelloworldClient(conn)

    // Create context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Make the call
    response, err := client.GetServerResponse(ctx, &pb.Message{
        Message: "Hello from Go client",
    })
    if err != nil {
        log.Fatalf("gRPC call failed: %v", err)
    }

    log.Printf("Response: %s", response.Message)
    log.Printf("Received: %t", response.Received)
}
```

### Advanced Go Client with Interceptors
```go
// advanced_client.go
package main

import (
    "context"
    "crypto/tls"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "google.golang.org/grpc/metadata"
    pb "path/to/your/proto/helloworld"
)

// Interceptor for logging
func loggingInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    start := time.Now()
    err := invoker(ctx, method, req, reply, cc, opts...)
    duration := time.Since(start)
    
    log.Printf("Method: %s, Duration: %v, Error: %v", method, duration, err)
    return err
}

type GrpcClient struct {
    client pb.HelloworldClient
    conn   *grpc.ClientConn
}

func NewGrpcClient(address string) (*GrpcClient, error) {
    config := &tls.Config{
        ServerName: "grpc-service.example.com",
    }
    creds := credentials.NewTLS(config)

    conn, err := grpc.Dial(address,
        grpc.WithTransportCredentials(creds),
        grpc.WithUnaryInterceptor(loggingInterceptor),
    )
    if err != nil {
        return nil, err
    }

    return &GrpcClient{
        client: pb.NewHelloworldClient(conn),
        conn:   conn,
    }, nil
}

func (c *GrpcClient) Close() error {
    return c.conn.Close()
}

func (c *GrpcClient) CallService(ctx context.Context, message string) (*pb.MessageResponse, error) {
    // Add metadata
    md := metadata.Pairs(
        "client-id", "go-advanced-client",
        "timestamp", time.Now().Format(time.RFC3339),
    )
    ctx = metadata.NewOutgoingContext(ctx, md)

    return c.client.GetServerResponse(ctx, &pb.Message{
        Message: message,
    })
}

func main() {
    client, err := NewGrpcClient("grpc-service.example.com:443")
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    defer client.Close()

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    response, err := client.CallService(ctx, "Hello from advanced Go client")
    if err != nil {
        log.Fatalf("gRPC call failed: %v", err)
    }

    log.Printf("Response: %s", response.Message)
    log.Printf("Received: %t", response.Received)
}
```

## Node.js Client Examples

### Basic Node.js Client
```javascript
// client.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// Load the protobuf
const PROTO_PATH = path.join(__dirname, 'helloworld.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
});

const helloworld = grpc.loadPackageDefinition(packageDefinition).helloworld;

function main() {
    // Create SSL credentials
    const credentials = grpc.credentials.createSsl();
    
    // Create client
    const client = new helloworld.helloworld(
        'grpc-service.example.com:443',
        credentials
    );

    // Make the call
    client.GetServerResponse(
        { message: 'Hello from Node.js client' },
        (error, response) => {
            if (error) {
                console.error('gRPC error:', error);
                return;
            }
            
            console.log('Response:', response.message);
            console.log('Received:', response.received);
        }
    );
}

main();
```

### Promise-based Node.js Client
```javascript
// async_client.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const { promisify } = require('util');

class GrpcClient {
    constructor(address) {
        const PROTO_PATH = './helloworld.proto';
        const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
            keepCase: true,
            longs: String,
            enums: String,
            defaults: true,
            oneofs: true
        });

        const helloworld = grpc.loadPackageDefinition(packageDefinition).helloworld;
        const credentials = grpc.credentials.createSsl();
        
        this.client = new helloworld.helloworld(address, credentials);
        this.getServerResponse = promisify(this.client.GetServerResponse.bind(this.client));
    }

    async callService(message) {
        try {
            const response = await this.getServerResponse({ message });
            return response;
        } catch (error) {
            console.error('gRPC error:', error);
            throw error;
        }
    }

    async healthCheck() {
        try {
            const response = await this.callService('Health check');
            return response.received;
        } catch {
            return false;
        }
    }
}

async function main() {
    const client = new GrpcClient('grpc-service.example.com:443');
    
    // Health check
    const isHealthy = await client.healthCheck();
    console.log('Service healthy:', isHealthy);
    
    if (!isHealthy) return;
    
    // Test various messages
    const testMessages = [
        'Hello World',
        'Testing Node.js client',
        'Unicode: 你好世界',
        ''
    ];
    
    for (const message of testMessages) {
        try {
            const response = await client.callService(message);
            console.log(`Success for "${message}":`, response.message.substring(0, 50) + '...');
        } catch (error) {
            console.error(`Failed for "${message}":`, error.message);
        }
    }
}

main().catch(console.error);
```

## Testing and Load Testing Examples

### Load Testing with grpcurl
```bash
#!/bin/bash
# load_test.sh

ENDPOINT="grpc-service.example.com:443"
CONCURRENT_CALLS=10
TOTAL_CALLS=100

echo "Running load test: $TOTAL_CALLS calls with $CONCURRENT_CALLS concurrent"

# Function to make gRPC call
make_call() {
    local id=$1
    grpcurl -d "{\"message\": \"Load test call $id\"}" \
        $ENDPOINT helloworld.helloworld/GetServerResponse > /dev/null 2>&1
    echo "Call $id completed"
}

# Track timing
start_time=$(date +%s)

# Run concurrent calls
for ((i=1; i<=TOTAL_CALLS; i++)); do
    make_call $i &
    
    # Limit concurrent calls
    if (( i % CONCURRENT_CALLS == 0 )); then
        wait
    fi
done

# Wait for remaining calls
wait

end_time=$(date +%s)
duration=$((end_time - start_time))

echo "Load test completed in ${duration}s"
echo "Average: $((TOTAL_CALLS / duration)) calls per second"
```

### Python Load Testing
```python
#!/usr/bin/env python3
# load_test.py

import asyncio
import time
import statistics
from concurrent.futures import ThreadPoolExecutor
import grpc
import helloworld_pb2
import helloworld_pb2_grpc

class LoadTester:
    def __init__(self, endpoint):
        self.endpoint = endpoint
        self.credentials = grpc.ssl_channel_credentials()
    
    def make_single_call(self, call_id):
        """Make a single gRPC call"""
        start_time = time.time()
        try:
            with grpc.secure_channel(self.endpoint, self.credentials) as channel:
                stub = helloworld_pb2_grpc.helloworldStub(channel)
                request = helloworld_pb2.Message(message=f"Load test {call_id}")
                response = stub.GetServerResponse(request, timeout=10)
                
                end_time = time.time()
                return {
                    'success': True,
                    'duration': (end_time - start_time) * 1000,  # ms
                    'call_id': call_id
                }
        except Exception as e:
            end_time = time.time()
            return {
                'success': False,
                'duration': (end_time - start_time) * 1000,  # ms
                'call_id': call_id,
                'error': str(e)
            }
    
    def run_load_test(self, total_calls=100, concurrent_calls=10):
        """Run load test with specified parameters"""
        print(f"Starting load test: {total_calls} calls, {concurrent_calls} concurrent")
        
        start_time = time.time()
        results = []
        
        with ThreadPoolExecutor(max_workers=concurrent_calls) as executor:
            futures = [
                executor.submit(self.make_single_call, i) 
                for i in range(total_calls)
            ]
            
            for future in futures:
                results.append(future.result())
        
        end_time = time.time()
        
        # Analyze results
        successful_calls = [r for r in results if r['success']]
        failed_calls = [r for r in results if not r['success']]
        
        durations = [r['duration'] for r in successful_calls]
        
        print(f"\nLoad Test Results:")
        print(f"Total time: {end_time - start_time:.2f}s")
        print(f"Total calls: {total_calls}")
        print(f"Successful: {len(successful_calls)}")
        print(f"Failed: {len(failed_calls)}")
        print(f"Success rate: {len(successful_calls)/total_calls*100:.1f}%")
        
        if durations:
            print(f"Response times (ms):")
            print(f"  Average: {statistics.mean(durations):.2f}")
            print(f"  Median: {statistics.median(durations):.2f}")
            print(f"  Min: {min(durations):.2f}")
            print(f"  Max: {max(durations):.2f}")
            print(f"  95th percentile: {statistics.quantiles(durations, n=20)[18]:.2f}")
        
        if failed_calls:
            print(f"\nFailure details:")
            for call in failed_calls[:5]:  # Show first 5 failures
                print(f"  Call {call['call_id']}: {call.get('error', 'Unknown error')}")

if __name__ == '__main__':
    tester = LoadTester('grpc-service.example.com:443')
    tester.run_load_test(total_calls=100, concurrent_calls=10) 
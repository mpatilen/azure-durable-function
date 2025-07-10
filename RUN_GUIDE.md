# Azure Durable Functions POC - Quick Run Guide

## 🚀 Quick Start

### Prerequisites

- Node.js 18+
- Azure Functions Core Tools v4
- Azurite (Azure Storage Emulator)

### Step 1: Install Dependencies

```bash
npm install
```

### Step 2: Build the Project

```bash
npm run build
```

### Step 3: Start Azurite (Terminal 1)

```bash
azurite --silent
```

### Step 4: Start Azure Functions (Terminal 2)

```bash
func start
```

### Step 5: Test the POC

```bash
node test-order-processing.js
```

## 📋 What This POC Demonstrates

- **Order Processing Workflow**: Inventory validation → Payment processing → Inventory update → Email & Shipping label (parallel)
- **State Management**: Automatic checkpointing and state persistence
- **Error Handling**: Built-in retry logic and error propagation
- **Monitoring**: Status polling and progress tracking

## 🔧 API Endpoints

- **Start Order Processing**: `POST http://localhost:7071/api/startOrderProcessing`
- **Check Status**: `GET http://localhost:7071/api/getStatus?instanceId={id}`

## 📊 Expected Output

The test script will:

1. Start an order processing orchestration
2. Poll for status updates every 2 seconds
3. Show the complete workflow execution
4. Display final results with transaction ID, email ID, and shipping label

## 🛠️ Troubleshooting

- **Port 7071 in use**: Kill existing func processes with `pkill -f "func start"`
- **Build errors**: Run `npm run clean && npm run build`
- **Azurite issues**: Restart with `azurite --silent`

## 📝 Demo Script

For team presentation, run:

```bash
# Terminal 1: Start services
azurite --silent &
func start &

# Terminal 2: Run demo
node test-order-processing.js
```

The POC is ready for your team presentation! 🎉

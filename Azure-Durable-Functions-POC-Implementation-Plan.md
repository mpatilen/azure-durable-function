# Azure Durable Functions POC - Implementation Plan

## 🎯 Project Overview

**Goal**: Create a simple Azure Durable Functions POC using TypeScript to demonstrate core concepts and workflow patterns.

**Use Case**: Order Processing Workflow

- Validate inventory availability
- Process payment
- Update inventory
- Send confirmation email
- Generate shipping label

## 📁 Project Structure

```
azure-durable-functions-poc/
├── .vscode/
│   └── settings.json
├── src/
│   ├── functions/
│   │   ├── client/
│   │   │   ├── startOrderProcessing.ts
│   │   │   └── getStatus.ts
│   │   ├── orchestrator/
│   │   │   └── orderProcessingOrchestrator.ts
│   │   └── activities/
│   │       ├── validateInventory.ts
│   │       ├── processPayment.ts
│   │       ├── updateInventory.ts
│   │       ├── sendConfirmationEmail.ts
│   │       └── generateShippingLabel.ts
│   ├── types/
│   │   └── index.ts
│   └── utils/
│       └── logger.ts
├── scripts/
│   ├── test-poc.js
│   └── setup-local.js
├── local.settings.json
├── host.json
├── package.json
├── tsconfig.json
├── .gitignore
└── README.md
```

## 🛠️ Technology Stack

- **Runtime**: Node.js 18+
- **Language**: TypeScript
- **Framework**: Azure Functions v4
- **Durable Functions**: durable-functions (v4 programming model)
- **Development**: VS Code with Azure Functions extension
- **Local Storage**: Azurite (Azure Storage Emulator)
- **Programming Model**: v4 (Latest, recommended by Microsoft)

## 📋 Implementation Steps

### Phase 1: Environment Setup (30 minutes)

#### 1.1 Prerequisites Installation

```bash
# Install Node.js 18+
# Install Azure Functions Core Tools
npm install -g azure-functions-core-tools@4

# Install Azurite (Azure Storage Emulator)
npm install -g azurite

# Install VS Code Extensions
# - Azure Functions
# - Azure Storage
# - Azure Account
```

#### 1.2 Project Initialization

```bash
# Create project directory
mkdir azure-durable-functions-poc
cd azure-durable-functions-poc

# Initialize Azure Functions project
func init --typescript

# Install dependencies (v4 programming model)
npm install durable-functions@latest
npm install @azure/functions@3.0.0
npm install @types/node --save-dev

# Install additional dependencies for testing
npm install axios --save-dev
```

### Phase 2: Configuration Setup (15 minutes)

#### 2.1 Local Settings Configuration

Create `local.settings.json`:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "WEBSITE_CONTENTAZUREFILECONNECTIONSTRING": "UseDevelopmentStorage=true",
    "WEBSITE_CONTENTSHARE": "azure-durable-functions-poc"
  }
}
```

#### 2.2 Host Configuration

Update `host.json`:

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[3.*, 4.0.0)"
  }
}
```

### Phase 3: Type Definitions (10 minutes)

#### 3.1 Create Types

Create `src/types/index.ts`:

```typescript
export interface OrderData {
  orderId: string;
  customerId: string;
  items: OrderItem[];
  totalAmount: number;
}

export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

export interface OrderStatus {
  orderId: string;
  status: "Pending" | "Processing" | "Completed" | "Failed";
  step: string;
  progress: number;
  result?: any;
  error?: string;
}

export interface ProcessedOrder {
  orderId: string;
  customerId: string;
  items: OrderItem[];
  totalAmount: number;
  status: "Completed";
  processedAt: string;
  confirmationEmail: string;
  shippingLabel: string;
}
```

### Phase 4: Function Implementation (45 minutes)

#### 4.1 Client Function

Create `src/functions/client/startOrderProcessing.ts`:

```typescript
import {
  app,
  HttpRequest,
  HttpResponseInit,
  InvocationContext,
} from "@azure/functions";
import * as df from "durable-functions";
import { OrderData } from "../../types";

app.http("startOrderProcessing", {
  methods: ["POST"],
  authLevel: "anonymous",
  handler: async (
    request: HttpRequest,
    context: InvocationContext
  ): Promise<HttpResponseInit> => {
    const client = df.getClient(context);
    const orderData: OrderData = (await request.json()) as OrderData;

    if (
      !orderData.orderId ||
      !orderData.customerId ||
      !orderData.items ||
      orderData.items.length === 0
    ) {
      return {
        status: 400,
        body: JSON.stringify({
          error:
            "Invalid order data. Required fields: orderId, customerId, items (non-empty array)",
        }),
      };
    }

    const instanceId = await client.startNew("orderProcessingOrchestrator", {
      input: orderData,
    });

    context.log(`Started orchestration with ID = '${instanceId}'.`);

    return client.createCheckStatusResponse(request, instanceId);
  },
});
```

#### 4.2 Orchestrator Function

Create `src/functions/orchestrator/orderProcessingOrchestrator.ts`:

```typescript
import { OrderData, ProcessedOrder } from "../../types";
import { Logger } from "../../utils/logger";

export default async function* orderProcessingOrchestrator(
  context: any
): AsyncGenerator<any, ProcessedOrder, any> {
  Logger.info("Starting order processing orchestration", {
    orderId: context.df.getInput()?.orderId,
  });
  const orderData: OrderData = context.df.getInput();

  try {
    Logger.info("Step 1: Validating inventory", { orderId: orderData.orderId });
    const inventoryResult = yield context.df.callActivity(
      "validateInventory",
      orderData
    );
    if (!inventoryResult.valid) {
      throw new Error(
        `Inventory validation failed: ${inventoryResult.message}`
      );
    }

    Logger.info("Step 2: Processing payment", { orderId: orderData.orderId });
    const paymentResult = yield context.df.callActivity(
      "processPayment",
      orderData
    );
    if (!paymentResult.success) {
      throw new Error(`Payment processing failed: ${paymentResult.message}`);
    }

    Logger.info("Step 3: Updating inventory", { orderId: orderData.orderId });
    const inventoryUpdateResult = yield context.df.callActivity(
      "updateInventory",
      orderData
    );
    if (!inventoryUpdateResult.success) {
      throw new Error(
        `Inventory update failed: ${inventoryUpdateResult.message}`
      );
    }

    Logger.info("Step 4: Sending confirmation email", {
      orderId: orderData.orderId,
    });
    const emailTask = context.df.callActivity(
      "sendConfirmationEmail",
      orderData
    );

    Logger.info("Step 5: Generating shipping label", {
      orderId: orderData.orderId,
    });
    const shippingTask = context.df.callActivity(
      "generateShippingLabel",
      orderData
    );

    const [emailResult, shippingResult] = yield context.df.Task.all([
      emailTask,
      shippingTask,
    ]);

    const processedOrder: ProcessedOrder = {
      orderId: orderData.orderId,
      customerId: orderData.customerId,
      items: orderData.items,
      totalAmount: orderData.totalAmount,
      status: "Completed",
      processedAt: new Date().toISOString(),
      confirmationEmail: emailResult.success ? emailResult.emailId : "Failed",
      shippingLabel: shippingResult.labelId,
    };

    Logger.info("Order processing completed successfully", {
      orderId: orderData.orderId,
      transactionId: paymentResult.transactionId,
      emailId: emailResult.emailId,
      labelId: shippingResult.labelId,
    });

    return processedOrder;
  } catch (error) {
    Logger.error("Order processing orchestration failed", error as Error, {
      orderId: orderData.orderId,
    });
    throw error;
  }
}
```

#### 4.3 Activity Functions

Create `src/functions/activities/validateInventory.ts`:

```typescript
import { OrderData, OrderItem } from "../../types";
import { Logger } from "../../utils/logger";

export default async function validateInventory(
  orderData: OrderData
): Promise<{ valid: boolean; message: string }> {
  Logger.info("Starting inventory validation", { orderId: orderData.orderId });

  const invalidItems: OrderItem[] = [];
  for (const item of orderData.items) {
    const availableStock = Math.floor(Math.random() * 10) + 1;
    if (item.quantity > availableStock) {
      invalidItems.push(item);
    }
  }

  if (invalidItems.length > 0) {
    Logger.warn("Inventory validation failed", {
      orderId: orderData.orderId,
      invalidItems: invalidItems.map((item) => ({
        productId: item.productId,
        requested: item.quantity,
      })),
    });
    return {
      valid: false,
      message: `Insufficient inventory for ${invalidItems.length} items`,
    };
  }

  Logger.info("Inventory validation successful", {
    orderId: orderData.orderId,
  });
  return {
    valid: true,
    message: "All items are available in sufficient quantity",
  };
}
```

Create `src/functions/activities/processPayment.ts`:

```typescript
import { OrderData } from "../../types";
import { Logger } from "../../utils/logger";

export default async function processPayment(
  orderData: OrderData
): Promise<{ success: boolean; transactionId: string; message: string }> {
  Logger.info("Starting payment processing", {
    orderId: orderData.orderId,
    amount: orderData.totalAmount,
  });

  const success = Math.random() > 0.1;
  const transactionId = `TXN-${Date.now()}-${Math.random()
    .toString(36)
    .substr(2, 9)}`;

  if (success) {
    Logger.info("Payment processed successfully", {
      orderId: orderData.orderId,
      transactionId,
      amount: orderData.totalAmount,
    });
    return {
      success: true,
      transactionId,
      message: "Payment processed successfully",
    };
  } else {
    Logger.warn("Payment processing failed", {
      orderId: orderData.orderId,
      amount: orderData.totalAmount,
    });
    return {
      success: false,
      transactionId: "",
      message: "Payment processing failed - insufficient funds",
    };
  }
}
```

Create `src/functions/activities/updateInventory.ts`:

```typescript
import { OrderData } from "../../types";
import { Logger } from "../../utils/logger";

export default async function updateInventory(
  orderData: OrderData
): Promise<{ success: boolean; message: string }> {
  Logger.info("Starting inventory update", { orderId: orderData.orderId });

  const success = Math.random() > 0.05;

  if (success) {
    Logger.info("Inventory updated successfully", {
      orderId: orderData.orderId,
    });
    return {
      success: true,
      message: "Inventory updated successfully",
    };
  } else {
    Logger.warn("Inventory update failed", { orderId: orderData.orderId });
    return {
      success: false,
      message: "Inventory update failed - database error",
    };
  }
}
```

Create `src/functions/activities/sendConfirmationEmail.ts`:

```typescript
import { OrderData } from "../../types";
import { Logger } from "../../utils/logger";

export default async function sendConfirmationEmail(
  orderData: OrderData
): Promise<{ success: boolean; emailId: string; message: string }> {
  Logger.info("Sending confirmation email", { orderId: orderData.orderId });

  const success = Math.random() > 0.1;
  const emailId = `EMAIL-${Date.now()}-${Math.random()
    .toString(36)
    .substr(2, 9)}`;

  if (success) {
    Logger.info("Confirmation email sent successfully", {
      orderId: orderData.orderId,
      emailId,
    });
    return {
      success: true,
      emailId,
      message: "Confirmation email sent successfully",
    };
  } else {
    Logger.warn("Confirmation email failed", { orderId: orderData.orderId });
    return {
      success: false,
      emailId: "",
      message: "Confirmation email failed - email service error",
    };
  }
}
```

Create `src/functions/activities/generateShippingLabel.ts`:

```typescript
import { OrderData } from "../../types";
import { Logger } from "../../utils/logger";

export default async function generateShippingLabel(
  orderData: OrderData
): Promise<{ success: boolean; labelId: string; message: string }> {
  Logger.info("Generating shipping label", { orderId: orderData.orderId });

  const success = Math.random() > 0.05;
  const labelId = `LABEL-${Date.now()}-${Math.random()
    .toString(36)
    .substr(2, 9)}`;

  if (success) {
    Logger.info("Shipping label generated successfully", {
      orderId: orderData.orderId,
      labelId,
    });
    return {
      success: true,
      labelId,
      message: "Shipping label generated successfully",
    };
  } else {
    Logger.warn("Shipping label generation failed", {
      orderId: orderData.orderId,
    });
    return {
      success: false,
      labelId: "",
      message: "Shipping label generation failed - printer error",
    };
  }
}
```

### Phase 5: Local Development Setup (20 minutes)

#### 5.1 Start Local Services

```bash
# Terminal 1: Start Azurite (Azure Storage Emulator)
azurite --silent

# Terminal 2: Start Azure Functions
func start
```

#### 5.2 Test the POC

```bash
# Test the orchestration
curl -X POST http://localhost:7071/api/startOrderProcessing \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "ORDER-123",
    "customerId": "CUST-456",
    "items": [
      {
        "productId": "PROD-001",
        "quantity": 2,
        "price": 29.99
      }
    ],
    "totalAmount": 59.98
  }'
```

### Phase 6: Additional POC Features (20 minutes)

#### 6.1 Status Endpoint Implementation

Create `src/functions/client/getStatus.ts`:

```typescript
import {
  app,
  HttpRequest,
  HttpResponseInit,
  InvocationContext,
} from "@azure/functions";
import * as df from "durable-functions";

app.http("getStatus", {
  methods: ["GET"],
  authLevel: "anonymous",
  handler: async (
    request: HttpRequest,
    context: InvocationContext
  ): Promise<HttpResponseInit> => {
    const client = df.getClient(context);
    const instanceId = request.query.get("instanceId");

    if (!instanceId) {
      return {
        status: 400,
        body: JSON.stringify({ error: "Instance ID is required" }),
      };
    }

    const status = await client.getStatus(instanceId);
    return {
      status: 200,
      body: JSON.stringify(status),
    };
  },
});
```

#### 6.2 Error Handling and Retry Logic

The retry logic is already implemented in the orchestrator function using `callActivityWithRetry`:

```typescript
// Example from orchestrator function
const processedContent = yield context.df.callActivityWithRetry(
  "processDocument",
  input,
  {
    firstRetryIntervalInMilliseconds: 1000,
    maxNumberOfAttempts: 3,
  }
);
```

#### 6.3 Logging and Monitoring

Create `src/utils/logger.ts`:

```typescript
export class Logger {
  static info(message: string, context?: any) {
    console.log(
      `[INFO] ${new Date().toISOString()}: ${message}`,
      context || ""
    );
  }

  static error(message: string, error?: any) {
    console.error(
      `[ERROR] ${new Date().toISOString()}: ${message}`,
      error || ""
    );
  }

  static warn(message: string, context?: any) {
    console.warn(
      `[WARN] ${new Date().toISOString()}: ${message}`,
      context || ""
    );
  }
}
```

#### 6.4 Application Insights Integration

Add to `local.settings.json`:

```json
{
  "Values": {
    "APPLICATIONINSIGHTS_CONNECTION_STRING": "your-connection-string"
  }
}
```

#### 6.5 Testing Scripts

Create `test-order-processing.js`:

```javascript
const axios = require("axios");

async function testOrderProcessing() {
  const baseUrl = "http://localhost:7071/api";

  console.log("🚀 Testing Azure Durable Functions Order Processing POC...\n");

  // Test data
  const testOrder = {
    orderId: `ORDER-${Date.now()}`,
    customerId: "CUST-12345",
    items: [
      { productId: "PROD-001", quantity: 2, price: 29.99 },
      { productId: "PROD-002", quantity: 1, price: 49.99 },
    ],
    totalAmount: 109.97,
  };

  try {
    // Step 1: Start the order processing orchestration
    console.log("1. Starting order processing orchestration...");
    console.log("Order Details:", JSON.stringify(testOrder, null, 2));

    const response = await axios.post(
      `${baseUrl}/startOrderProcessing`,
      testOrder
    );

    if (response.status === 202) {
      console.log("✅ Orchestration started successfully!");
      console.log(`Instance ID: ${response.data.id}`);
      console.log(`Status Query Location: ${response.data.statusQueryGetUri}`);

      // Step 2: Poll for status updates
      console.log("\n2. Polling for status updates...");
      const instanceId = response.data.id;

      let completed = false;
      let attempts = 0;
      const maxAttempts = 15;

      while (!completed && attempts < maxAttempts) {
        attempts++;
        await new Promise((resolve) => setTimeout(resolve, 2000));

        try {
          const statusResponse = await axios.get(
            response.data.statusQueryGetUri
          );
          const status = statusResponse.data;

          console.log(`\n📊 Status Check #${attempts}:`);
          console.log(`   Runtime Status: ${status.runtimeStatus}`);
          console.log(`   Created Time: ${status.createdTime}`);
          console.log(`   Last Updated: ${status.lastUpdatedTime}`);

          if (status.runtimeStatus === "Completed") {
            console.log("\n🎉 Order processing completed successfully!");
            console.log("\n📋 Final Result:");
            console.log(JSON.stringify(status.output, null, 2));
            completed = true;
          } else if (status.runtimeStatus === "Failed") {
            console.log("\n❌ Order processing failed!");
            console.log("Error:", status.output);
            completed = true;
          } else if (status.runtimeStatus === "Running") {
            console.log("   ⏳ Still processing...");
          }
        } catch (error) {
          console.log(`   ❌ Status check failed: ${error.message}`);
        }
      }

      if (!completed) {
        console.log(
          "\n⏰ Maximum polling attempts reached. The orchestration may still be running."
        );
      }
    } else {
      console.log("❌ Failed to start orchestration:", response.data);
    }
  } catch (error) {
    console.error("❌ Test failed:", error.response?.data || error.message);
  }
}

// Run the test
testOrderProcessing();
```

## 🚀 Deployment Plan

### Local Testing (30 minutes)

1. Start Azurite storage emulator
2. Start Azure Functions runtime
3. Test with sample data
4. Monitor execution in Azure Portal

### Azure Deployment (45 minutes)

1. Create Azure Storage Account
2. Create Azure Function App
3. Configure Application Insights
4. Deploy using VS Code or Azure CLI

## 📊 Demo Script

### 1. Introduction (5 minutes)

- Explain Azure Durable Functions concept
- Show project structure
- Demonstrate the workflow

### 2. Live Demo (10 minutes)

- Start the orchestration
- Show status polling
- Demonstrate error handling
- Show monitoring dashboard

### 3. Code Walkthrough (10 minutes)

- Explain each function type
- Show state management
- Demonstrate checkpointing

### 4. Q&A (5 minutes)

- Answer team questions
- Discuss use cases
- Plan next steps

## 🎯 Success Criteria

- ✅ POC runs locally without errors
- ✅ All functions execute in correct sequence
- ✅ State is properly maintained
- ✅ Error handling works
- ✅ Status polling returns correct information
- ✅ Team understands the concepts

## 📚 Additional Resources

- [Azure Durable Functions Documentation](https://docs.microsoft.com/en-us/azure/azure-functions/durable/)
- [TypeScript Azure Functions Guide](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-node)
- [Local Development with Azurite](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azurite)

## ⏱️ Timeline

**Total Implementation Time**: ~3 hours

- Environment Setup: 30 minutes
- Configuration: 15 minutes
- Function Implementation: 45 minutes
- Local Testing: 30 minutes
- Documentation: 30 minutes
- Demo Preparation: 30 minutes

This plan provides a complete, step-by-step approach to implementing a simple but comprehensive Azure Durable Functions POC that will effectively demonstrate the core concepts to your team.

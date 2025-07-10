# Azure Durable Functions POC - Implementation Plan

## 🎯 Project Overview

**Goal**: Create a simple Azure Durable Functions POC using TypeScript to demonstrate core concepts and workflow patterns.

**Use Case**: Document Processing Workflow

- Upload document
- Process document (simulate OCR)
- Validate data
- Store results
- Send notification

## 📁 Project Structure

```
azure-durable-functions-poc/
├── .vscode/
│   └── settings.json
├── src/
│   ├── functions/
│   │   ├── client/
│   │   │   ├── startOrchestration.ts
│   │   │   └── getStatus.ts
│   │   ├── orchestrator/
│   │   │   └── documentProcessingOrchestrator.ts
│   │   └── activities/
│   │       ├── processDocument.ts
│   │       ├── validateData.ts
│   │       ├── storeResults.ts
│   │       └── sendNotification.ts
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
export interface DocumentProcessingInput {
  documentId: string;
  documentName: string;
  documentContent: string;
}

export interface ProcessingResult {
  documentId: string;
  processedContent: string;
  validationStatus: "valid" | "invalid";
  timestamp: Date;
}

export interface ValidationResult {
  isValid: boolean;
  errors: string[];
}

export interface NotificationData {
  recipient: string;
  message: string;
  status: "success" | "error";
}

export interface OrchestrationResult {
  documentId: string;
  status: "completed" | "failed";
  timestamp: string;
  processedContent?: string;
  validationStatus?: string;
}
```

### Phase 4: Function Implementation (45 minutes)

#### 4.1 Client Function

Create `src/functions/client/startOrchestration.ts`:

```typescript
import {
  app,
  HttpRequest,
  HttpResponseInit,
  InvocationContext,
} from "@azure/functions";
import { DurableClient } from "durable-functions";
import { DocumentProcessingInput } from "../../types";

app.http("startOrchestration", {
  methods: ["POST"],
  authLevel: "anonymous",
  handler: async (
    request: HttpRequest,
    context: InvocationContext
  ): Promise<HttpResponseInit> => {
    const client = new DurableClient(context);
    const body = (await request.json()) as DocumentProcessingInput;
    const { documentId, documentName, documentContent } = body;

    if (!documentId || !documentName || !documentContent) {
      return {
        status: 400,
        body: JSON.stringify({
          error:
            "Missing required fields: documentId, documentName, documentContent",
        }),
      };
    }

    const instanceId = await client.startNew("documentProcessingOrchestrator", {
      documentId,
      documentName,
      documentContent,
    });

    context.log(`Started orchestration with ID = '${instanceId}'.`);

    return client.createCheckStatusResponse(request, instanceId);
  },
});
```

#### 4.2 Orchestrator Function

Create `src/functions/orchestrator/documentProcessingOrchestrator.ts`:

```typescript
import { app, DurableOrchestrationContext } from "@azure/functions";
import { DocumentProcessingInput, OrchestrationResult } from "../../types";

app.orchestration(
  "documentProcessingOrchestrator",
  async function* (
    context: DurableOrchestrationContext
  ): AsyncGenerator<any, OrchestrationResult> {
    try {
      const input = context.df.getInput() as DocumentProcessingInput;
      context.log(`Starting document processing for: ${input.documentName}`);

      // Step 1: Process Document with retry logic
      const processedContent = yield context.df.callActivityWithRetry(
        "processDocument",
        input,
        {
          firstRetryIntervalInMilliseconds: 1000,
          maxNumberOfAttempts: 3,
        }
      );
      context.log("Document processed successfully");

      // Step 2: Validate Data with retry logic
      const validationResult = yield context.df.callActivityWithRetry(
        "validateData",
        {
          documentId: input.documentId,
          processedContent,
        },
        {
          firstRetryIntervalInMilliseconds: 500,
          maxNumberOfAttempts: 2,
        }
      );
      context.log("Data validation completed");

      // Step 3: Store Results
      const storageResult = yield context.df.callActivity("storeResults", {
        documentId: input.documentId,
        processedContent,
        validationStatus: validationResult.isValid ? "valid" : "invalid",
      });
      context.log("Results stored successfully");

      // Step 4: Send Notification
      yield context.df.callActivity("sendNotification", {
        recipient: "admin@company.com",
        message: `Document ${
          input.documentName
        } processing completed with status: ${
          validationResult.isValid ? "SUCCESS" : "FAILED"
        }`,
        status: validationResult.isValid ? "success" : "error",
      });
      context.log("Notification sent");

      return {
        documentId: input.documentId,
        status: "completed",
        timestamp: new Date().toISOString(),
        processedContent,
        validationStatus: validationResult.isValid ? "valid" : "invalid",
      };
    } catch (error) {
      context.log(
        `Orchestration failed: ${
          error instanceof Error ? error.message : String(error)
        }`
      );
      throw error;
    }
  }
);
```

#### 4.3 Activity Functions

Create `src/functions/activities/processDocument.ts`:

```typescript
import { app } from "@azure/functions";
import { DocumentProcessingInput } from "../../types";

app.activity(
  "processDocument",
  async (input: DocumentProcessingInput): Promise<string> => {
    console.log(`Processing document: ${input.documentName}`);
    await new Promise((resolve) => setTimeout(resolve, 2000));
    const processedContent = `PROCESSED: ${input.documentContent.toUpperCase()}`;
    console.log(
      `Document processed successfully. Content length: ${processedContent.length}`
    );
    return processedContent;
  }
);
```

Create `src/functions/activities/validateData.ts`:

```typescript
import { app } from "@azure/functions";
import { ValidationResult } from "../../types";

app.activity(
  "validateData",
  async (input: {
    documentId: string;
    processedContent: string;
  }): Promise<ValidationResult> => {
    console.log(`Validating data for document: ${input.documentId}`);
    await new Promise((resolve) => setTimeout(resolve, 1000));

    const errors: string[] = [];

    if (!input.processedContent || input.processedContent.length < 10) {
      errors.push("Content too short - minimum 10 characters required");
    }

    if (!input.documentId) {
      errors.push("Invalid document ID");
    }

    if (!input.processedContent.includes("PROCESSED:")) {
      errors.push("Content was not properly processed");
    }

    const result: ValidationResult = {
      isValid: errors.length === 0,
      errors,
    };

    console.log(
      `Validation completed. Valid: ${result.isValid}, Errors: ${errors.length}`
    );
    return result;
  }
);
```

Create `src/functions/activities/storeResults.ts`:

```typescript
import { app } from "@azure/functions";
import { ProcessingResult } from "../../types";

app.activity(
  "storeResults",
  async (input: {
    documentId: string;
    processedContent: string;
    validationStatus: string;
  }): Promise<ProcessingResult> => {
    console.log(`Storing results for document: ${input.documentId}`);
    await new Promise((resolve) => setTimeout(resolve, 1000));

    const result: ProcessingResult = {
      documentId: input.documentId,
      processedContent: input.processedContent,
      validationStatus: input.validationStatus as "valid" | "invalid",
      timestamp: new Date(),
    };

    console.log(
      `Results stored successfully for document: ${input.documentId}`
    );
    console.log(`Storage timestamp: ${result.timestamp.toISOString()}`);
    return result;
  }
);
```

Create `src/functions/activities/sendNotification.ts`:

```typescript
import { app } from "@azure/functions";
import { NotificationData } from "../../types";

app.activity(
  "sendNotification",
  async (input: NotificationData): Promise<void> => {
    console.log(`Sending notification to: ${input.recipient}`);
    await new Promise((resolve) => setTimeout(resolve, 500));

    console.log(`📧 Notification sent to ${input.recipient}:`);
    console.log(`   Message: ${input.message}`);
    console.log(`   Status: ${input.status}`);
    // In a real implementation, you would send email/SMS/etc.
  }
);
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
curl -X POST http://localhost:7071/api/startOrchestration \
  -H "Content-Type: application/json" \
  -d '{
    "documentId": "doc-123",
    "documentName": "sample-document.txt",
    "documentContent": "This is a sample document for processing."
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
import { DurableClient } from "durable-functions";

app.http("getStatus", {
  methods: ["GET"],
  authLevel: "anonymous",
  handler: async (
    request: HttpRequest,
    context: InvocationContext
  ): Promise<HttpResponseInit> => {
    const client = new DurableClient(context);
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

Create `scripts/test-poc.js`:

```javascript
const axios = require("axios");

async function testPOC() {
  const baseUrl = "http://localhost:7071/api";

  console.log("🚀 Testing Azure Durable Functions POC...\n");

  try {
    // Start orchestration
    console.log("1. Starting document processing orchestration...");
    const response = await axios.post(`${baseUrl}/startOrchestration`, {
      documentId: "doc-123",
      documentName: "sample-document.txt",
      documentContent:
        "This is a sample document for processing demonstration.",
    });

    console.log("✅ Orchestration started successfully");
    console.log(`Instance ID: ${response.data.instanceId}\n`);

    // Poll for status
    console.log("2. Polling for status updates...");
    const instanceId = response.data.instanceId;

    for (let i = 0; i < 10; i++) {
      await new Promise((resolve) => setTimeout(resolve, 2000));

      try {
        const statusResponse = await axios.get(
          `${baseUrl}/getStatus?instanceId=${instanceId}`
        );
        const status = statusResponse.data;

        console.log(`Status check ${i + 1}: ${status.runtimeStatus}`);

        if (status.runtimeStatus === "Completed") {
          console.log("✅ Orchestration completed successfully!");
          console.log("Final result:", status.output);
          break;
        } else if (status.runtimeStatus === "Failed") {
          console.log("❌ Orchestration failed:", status.output);
          break;
        }
      } catch (error) {
        console.log(`Status check ${i + 1}: Error - ${error.message}`);
      }
    }
  } catch (error) {
    console.error("❌ Test failed:", error.response?.data || error.message);
  }
}

testPOC();
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

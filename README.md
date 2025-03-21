# AWS Demo with Amplify for Text Extraction

# DO NOT FOLLOW THIS DEMO: OUTDATED

This guide demonstrates how to build a complete AWS Amplify application that:
1. Provides a web interface for users to upload images
2. Triggers a Lambda function to process the image using AWS Textract
3. Returns the extracted text to the user
4. Sets up a complete CI/CD pipeline using GitHub and AWS Amplify

## Architecture Overview

The overall flow and interaction are:
1. A user interacts with the React frontend hosted on AWS Amplify
2. The frontend authenticates with Amazon Cognito
3. When an image is uploaded, it's stored in S3
4. The frontend calls the GraphQL API (AppSync)
5. AppSync triggers the Java Lambda function
6. The Lambda function retrieves the image from S3
7. The Lambda function calls AWS Textract to process the image
8. Results are stored in DynamoDB and returned to the frontend through AppSync

- **Frontend**: React web application hosted on AWS Amplify
- **Backend**: 
  - AWS Amplify API (AppSync GraphQL)
  - Lambda function (Java) for image processing
  - S3 bucket for image storage
  - AWS Textract for text extraction
- **CI/CD**: GitHub + AWS Amplify Hosting

This architecture demonstrates the power of serverless development with AWS Amplify - you can focus on building features while AWS handles the infrastructure scaling and management. And we will also experience the resource management through AWS CDK, SDK and CLI.

## Prerequisites

- AWS Account
- GitHub Account
- Windows 10+
- VS Code
- The following software installed:
  - Node.js & npm
  - AWS CLI
  - AWS Amplify CLI
  - Git
  - Java 11+
  - Maven (or integrated with VS Code)

## 1. Environment Setup

### 1.1 Install Required Software

```bash
# Install AWS CLI,  if not already installed
# curl "https://awscli.amazonaws.com/AWSCLIV2.msi" -o "AWSCLIV2.msi"
# msiexec.exe /i AWSCLIV2.msi

# Install AWS Amplify CLI,  if not already installed
# npm install -g @aws-amplify/cli

# Install Git if not already installed
# Download from https://git-scm.com/download/win

# Install Java 11+ (Amazon Corretto)
# Download from https://docs.aws.amazon.com/corretto/latest/corretto-11-ug/downloads-list.html

# Install Maven
# Download from https://maven.apache.org/download.cgi
```

### 1.2 Configure AWS CLI

```bash
aws configure
# Enter your AWS Access Key ID, Secret Access Key, region (e.g., us-east-1), and output format (json)
```

### 1.3 Initialize Amplify CLI

```bash
amplify configure
# Follow the prompts to set up a new IAM user for Amplify
```

## 2. Create Frontend Application

### 2.1 Create a React Application

```bash
npx create-react-app amplify-textract-app
cd amplify-textract-app
```

### 2.2 Initialize Amplify in the Project

```bash
amplify init
```

Follow the prompts:
- Enter a name for the project: `amplifyTextractApp`
- Enter a name for the environment: `dev`
- Choose your default editor: `Visual Studio Code`
- Choose the type of app: `javascript`
- Framework: `react`
- Source directory: `src`
- Distribution directory: `build`
- Build command: `npm run-script build`
- Start command: `npm run-script start`
- Select the authentication method: `AWS profile`
- Choose the profile you created during `amplify configure`

## 3. Add Authentication

```bash
amplify add auth
```

Follow the prompts:
- Default configuration: `Default configuration`
- How do you want users to be able to sign in: `Username`
- Do you want to configure advanced settings: `No, I am done`

## 4. Add Storage (S3)

```bash
amplify add storage
```

Follow the prompts:
- Select from the options: `Content (Images, audio, video, etc.)`
- Resource name: `textractimages`
- Bucket name: `amplify-textract-images`
- Who should have access: `Auth users only`
- What kind of access: `create/update, read, delete`
- Do you want to add a Lambda Trigger: `No`

## 5. Add GraphQL API

```bash
amplify add api
```

Follow the prompts:
- Select from the options: `GraphQL`
- API name: `textractapi`
- Authorization type: `Amazon Cognito User Pool`
- Do you want to configure advanced settings: `Yes`
- Configure additional auth types: `No`
- Configure conflict detection: `No`
- Do you want to use the default editor: `Yes`

Replace the contents of the schema.graphql file with:

```graphql
type TextractResult @model @auth(rules: [{allow: owner}]) {
  id: ID!
  imageKey: String!
  extractedText: String
  status: String!
  owner: String
}

type Mutation {
  processImage(imageKey: String!): TextractResult @function(name: "textractProcessor-${env}")
}
```

## 6. Create Lambda Function for Textract

### 6.1 Add Lambda Function

```bash
amplify add function
```

Follow the prompts:
- Select function template: `Lambda function triggerable by other AWS services`
- Name: `textractProcessor`
- Runtime: `java`
- Function name: `textractProcessor`
- Lambda access to other resources: `Yes`
- Select the categories: `api, storage`
- Select the operations: `create, read, update, delete` (for both)

### 6.2 Add Textract Permissions

Update the IAM policy for the Lambda function:

```bash
amplify update function
```

Select the function `textractProcessor` and choose to edit the local Lambda function's execution role.

Add the following policy to the function's IAM role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "textract:DetectDocumentText",
                "textract:AnalyzeDocument"
            ],
            "Resource": "*"
        }
    ]
}
```

### 6.3 Implement the Lambda Function in Java

Navigate to `amplify/backend/function/textractProcessor/src/main/java/example/`

Create a file named `LambdaHandler.java`:

```java
package example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.AppSyncLambdaAuthorizationEvent;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.GetObjectRequest;
import com.amazonaws.services.s3.model.S3Object;
import com.amazonaws.services.textract.AmazonTextract;
import com.amazonaws.services.textract.AmazonTextractClientBuilder;
import com.amazonaws.services.textract.model.DetectDocumentTextRequest;
import com.amazonaws.services.textract.model.DetectDocumentTextResult;
import com.amazonaws.services.textract.model.Document;
import com.amazonaws.services.textract.model.Block;
import com.amazonaws.services.textract.model.S3Object.Builder;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.InputStream;
import java.nio.ByteBuffer;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

public class LambdaHandler implements RequestHandler<Map<String, Object>, Map<String, Object>> {
    private static final Logger logger = LoggerFactory.getLogger(LambdaHandler.class);
    private static final String S3_BUCKET_NAME = System.getenv("STORAGE_TEXTRACTIMAGES_BUCKETNAME");
    private static final AmazonS3 s3Client = AmazonS3ClientBuilder.standard().build();
    private static final AmazonTextract textractClient = AmazonTextractClientBuilder.standard().build();
    private static final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Map<String, Object> handleRequest(Map<String, Object> input, Context context) {
        logger.info("Received event: {}", input);

        try {
            // Extract imageKey from request
            Map<String, Object> arguments = (Map<String, Object>) input.get("arguments");
            String imageKey = (String) arguments.get("imageKey");
            String owner = extractOwnerFromEvent(input);
            
            // Create result object
            Map<String, Object> result = new HashMap<>();
            result.put("id", UUID.randomUUID().toString());
            result.put("imageKey", imageKey);
            result.put("status", "PROCESSING");
            result.put("owner", owner);

            // Process the image with Textract
            String extractedText = extractTextFromImage(imageKey);
            result.put("extractedText", extractedText);
            result.put("status", "COMPLETED");

            return result;
        } catch (Exception e) {
            logger.error("Error processing image", e);
            Map<String, Object> errorResult = new HashMap<>();
            errorResult.put("id", UUID.randomUUID().toString());
            errorResult.put("imageKey", input.containsKey("arguments") ? 
                ((Map<String, Object>)input.get("arguments")).get("imageKey") : "unknown");
            errorResult.put("status", "ERROR");
            errorResult.put("extractedText", "Error processing image: " + e.getMessage());
            return errorResult;
        }
    }

    private String extractOwnerFromEvent(Map<String, Object> input) {
        try {
            Map<String, Object> identity = (Map<String, Object>) 
                ((Map<String, Object>) input.get("identity")).get("claims");
            return (String) identity.get("username");
        } catch (Exception e) {
            logger.warn("Could not extract owner from event", e);
            return "unknown";
        }
    }

    private String extractTextFromImage(String imageKey) throws IOException {
        // Get the image from S3
        S3Object s3Object = s3Client.getObject(new GetObjectRequest(S3_BUCKET_NAME, imageKey));
        
        // Prepare the document
        ByteBuffer imageBytes;
        try (InputStream is = s3Object.getObjectContent()) {
            imageBytes = ByteBuffer.wrap(is.readAllBytes());
        }
        
        // Create document for Textract
        Document document = new Document().withBytes(imageBytes);
        
        // Detect text
        DetectDocumentTextRequest request = new DetectDocumentTextRequest()
            .withDocument(document);
        
        DetectDocumentTextResult result = textractClient.detectDocumentText(request);
        
        // Extract text from blocks
        return result.getBlocks().stream()
            .filter(block -> "LINE".equals(block.getBlockType()))
            .map(Block::getText)
            .collect(Collectors.joining("\n"));
    }
}
```

Update the `build.gradle` file to include the necessary dependencies:

```gradle
dependencies {
    implementation 'com.amazonaws:aws-lambda-java-core:1.2.1'
    implementation 'com.amazonaws:aws-lambda-java-events:3.11.0'
    implementation 'com.amazonaws:aws-java-sdk-s3:1.12.112'
    implementation 'com.amazonaws:aws-java-sdk-textract:1.12.112'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.13.0'
    implementation 'org.slf4j:slf4j-api:1.7.32'
    implementation 'org.slf4j:slf4j-simple:1.7.32'
    testImplementation 'junit:junit:4.13.2'
}
```

## 7. Connect the Frontend

### 7.1 Install Amplify Libraries

```bash
npm install aws-amplify @aws-amplify/ui-react
```

### 7.2 Configure Amplify in the Frontend

Update `src/index.js`:

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';
import reportWebVitals from './reportWebVitals';
import { Amplify } from 'aws-amplify';
import awsExports from './aws-exports';

Amplify.configure(awsExports);

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);

reportWebVitals();
```

### 7.3 Create the Main App Component

Update `src/App.js`:

```jsx
import React, { useState, useEffect } from 'react';
import { withAuthenticator } from '@aws-amplify/ui-react';
import { Storage, API, graphqlOperation } from 'aws-amplify';
import '@aws-amplify/ui-react/styles.css';
import './App.css';

// GraphQL mutations and queries
const processImage = /* GraphQL */ `
  mutation ProcessImage($imageKey: String!) {
    processImage(imageKey: $imageKey) {
      id
      imageKey
      extractedText
      status
      owner
    }
  }
`;

const listTextractResults = /* GraphQL */ `
  query ListTextractResults {
    listTextractResults {
      items {
        id
        imageKey
        extractedText
        status
        owner
        createdAt
      }
    }
  }
`;

function App({ signOut, user }) {
  const [file, setFile] = useState(null);
  const [processing, setProcessing] = useState(false);
  const [results, setResults] = useState([]);
  const [errorMessage, setErrorMessage] = useState('');

  useEffect(() => {
    fetchResults();
  }, []);

  async function fetchResults() {
    try {
      const resultsData = await API.graphql(graphqlOperation(listTextractResults));
      setResults(resultsData.data.listTextractResults.items);
    } catch (error) {
      console.error('Error fetching results:', error);
      setErrorMessage('Error fetching previous results');
    }
  }

  async function handleFileChange(e) {
    const selectedFile = e.target.files[0];
    if (selectedFile) {
      setFile(selectedFile);
      setErrorMessage('');
    }
  }

  async function handleSubmit(e) {
    e.preventDefault();
    if (!file) {
      setErrorMessage('Please select a file first');
      return;
    }

    setProcessing(true);
    setErrorMessage('');

    try {
      // Upload file to S3
      const uniqueKey = `${user.username}/${Date.now()}-${file.name}`;
      await Storage.put(uniqueKey, file, {
        contentType: file.type,
      });

      // Process the image with Textract via Lambda
      const result = await API.graphql(graphqlOperation(processImage, { imageKey: uniqueKey }));
      
      // Update the UI with the result
      setResults([result.data.processImage, ...results]);
      setFile(null);
      document.getElementById('file-upload').value = '';
    } catch (error) {
      console.error('Error processing image:', error);
      setErrorMessage('Error processing image. Please try again.');
    } finally {
      setProcessing(false);
    }
  }

  return (
    <div className="App">
      <header className="App-header">
        <h1>AWS Amplify Textract Demo</h1>
        <p>Welcome, {user.username}</p>
        <button onClick={signOut}>Sign Out</button>
      </header>
      
      <main>
        <div className="upload-section">
          <h2>Upload Image for Text Extraction</h2>
          <form onSubmit={handleSubmit}>
            <div className="form-group">
              <label htmlFor="file-upload">Select Image:</label>
              <input
                id="file-upload"
                type="file"
                accept="image/*"
                onChange={handleFileChange}
                disabled={processing}
              />
            </div>
            
            {errorMessage && <p className="error">{errorMessage}</p>}
            
            <button type="submit" disabled={!file || processing}>
              {processing ? 'Processing...' : 'Extract Text'}
            </button>
          </form>
        </div>
        
        <div className="results-section">
          <h2>Extracted Text Results</h2>
          {results.length === 0 ? (
            <p>No results yet. Upload an image to extract text.</p>
          ) : (
            <div className="results-list">
              {results.map((result) => (
                <div key={result.id} className="result-item">
                  <h3>Image: {result.imageKey.split('/').pop()}</h3>
                  <p>Status: {result.status}</p>
                  <div className="text-box">
                    <h4>Extracted Text:</h4>
                    <pre>{result.extractedText || 'No text extracted'}</pre>
                  </div>
                  <p className="timestamp">Processed: {new Date(result.createdAt).toLocaleString()}</p>
                </div>
              ))}
            </div>
          )}
        </div>
      </main>
    </div>
  );
}

export default withAuthenticator(App);
```

### 7.4 Add Some Styling

Create `src/App.css`:

```css
.App {
  font-family: Arial, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.App-header {
  background-color: #282c34;
  color: white;
  padding: 20px;
  border-radius: 5px;
  margin-bottom: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.App-header h1 {
  margin: 0;
}

.App-header button {
  background-color: #61dafb;
  border: none;
  color: #282c34;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
}

main {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 20px;
}

.upload-section {
  background-color: #f5f5f5;
  padding: 20px;
  border-radius: 5px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
}

button {
  background-color: #282c34;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.error {
  color: red;
}

.results-section {
  background-color: #f5f5f5;
  padding: 20px;
  border-radius: 5px;
}

.result-item {
  background-color: white;
  padding: 15px;
  margin-bottom: 15px;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.text-box {
  background-color: #f9f9f9;
  padding: 10px;
  border-radius: 5px;
  margin: 10px 0;
}

pre {
  white-space: pre-wrap;
  word-wrap: break-word;
}

.timestamp {
  color: #666;
  font-size: 0.8em;
  text-align: right;
}

@media (max-width: 768px) {
  main {
    grid-template-columns: 1fr;
  }
}
```

## 8. Deploy the Application

### 8.1 Push the Backend to AWS

```bash
amplify push
```

Confirm that you want to generate the GraphQL API code and other resources.

### 8.2 Create a GitHub Repository

1. Create a new repository on GitHub
2. Initialize your local repository and push the code:

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/yourusername/amplify-textract-app.git
git push -u origin main
```

### 8.3 Configure Amplify Hosting with CI/CD

1. Log in to the AWS Management Console
2. Navigate to AWS Amplify
3. Click "New app" > "Host web app"
4. Choose GitHub as the repository provider
5. Connect to your GitHub account and select your repository
6. Choose the main branch
7. Review the build settings (Amplify will automatically detect that this is a React app)
8. Click "Save and deploy"

AWS Amplify will now:
1. Clone your GitHub repository
2. Build your application
3. Deploy it to a global CDN

### 8.4 Configure Amplify Backend with CI/CD

For the backend CI/CD configuration, create a new file `.github/workflows/backend-deploy.yml`:

```yaml
name: Deploy Backend

on:
  push:
    branches:
      - main
    paths:
      - 'amplify/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '14'
          
      - name: Set up Java
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt'
          java-version: '11'
          
      - name: Install Amplify CLI
        run: npm install -g @aws-amplify/cli
        
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
          
      - name: Deploy Amplify project
        run: |
          amplify env checkout dev
          amplify push --yes
```

Add the following secrets to your GitHub repository:
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_REGION

## 9. Testing the Application

1. Open the URL provided by Amplify Hosting
2. Create an account and log in
3. Upload an image containing text
4. Click "Extract Text" button
5. View the extracted text results

## 10. Understanding the Architecture

### How Amplify App, API, and Lambda Are Connected

1. **Amplify App (Frontend)**
   - Hosted on AWS Amplify
   - Uses Amplify libraries to interact with backend services
   - Authenticates users with Cognito
   - Stores images in S3
   - Calls GraphQL API to trigger Lambda functions

2. **Amplify API (AppSync GraphQL)**
   - Provides a GraphQL endpoint for the frontend
   - Handles authentication and authorization
   - Connects to Lambda functions and DynamoDB
   - Manages the data model and relationships

3. **Lambda Functions**
   - Process images using AWS Textract
   - Store results in DynamoDB
   - Connect to other AWS services (S3, Textract)

4. **CI/CD Pipeline**
   - GitHub triggers Amplify Hosting deployments for frontend
   - GitHub Actions triggers Amplify CLI deployments for backend
   - Ensures consistent deployment of both frontend and backend

## 11. Common Issues and Troubleshooting

1. **Authentication Issues**
   - Check Cognito user pool settings
   - Verify that the frontend is correctly configured with aws-exports.js

2. **Lambda Function Errors**
   - Check CloudWatch logs for the Lambda function
   - Verify IAM permissions for S3, Textract, and DynamoDB

3. **API Connection Issues**
   - Ensure GraphQL schema is correctly defined
   - Check for typos in GraphQL queries and mutations

4. **Deployment Failures**
   - Check build logs in Amplify Console
   - Verify GitHub Actions workflow configuration

## 12. Next Steps

1. **Enhance the UI**
   - Add loading indicators
   - Implement pagination for results
   - Add image preview functionality

2. **Improve Text Extraction**
   - Use Textract's AnalyzeDocument API for more advanced features
   - Implement forms processing capabilities

3. **Add More Features**
   - Implement search functionality
   - Add support for PDF documents
   - Enable sharing of extracted text

## Conclusion

This demo has demonstrated how AWS Amplify app, API, and Lambda functions work together to create a complete serverless application with CI/CD. You've seen how to:

- Create a React frontend with Amplify libraries
- Set up authentication using Cognito
- Store files in S3
- Process images with Lambda and Textract
- Create a GraphQL API with AppSync
- Deploy the application with a CI/CD pipeline

The integration between these components allows for a scalable, serverless architecture that can handle complex workflows while maintaining security and performance.

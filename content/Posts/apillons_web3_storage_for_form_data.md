+++
title = 'Apillons_web3_storage_for_form_data'
date = 2024-08-01T17:22:05+02:00
+++

## Introduction

- Brief introduction to the technologies used (Vite, React, TypeScript, Apillon Hosting, REST API).
- The purpose of the tutorial: to create a web form that saves data to Apillon Hosting.

## Setting Up the Project

### Creating the Project with Vite and TypeScript

First, let's create a new project using Vite with TypeScript.

Open your terminal and run the following commands:

```bash
npm create vite@latest my-react-app --template react-ts
cd my-react-app
```

### Updating package.json Scripts

Next, update your package.json to include a script for starting the JSON Server. Your scripts section should look like this:

```bash
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "preview": "vite preview"
  },
```

## Creating the Form Component

Now, let's create our form component in React.

### Creating Form.tsx

Create a file named Form.tsx in the src directory with the following content:

```bash
import React, { useState } from "react";

const Form: React.FC = () => {
  const [email, setEmail] = useState<string>("");
  const [crypto, setCrypto] = useState<string>("");

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    const formData = { email, crypto };

    try {
      const response = await fetch("http://localhost:5000/form", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(formData),
      });
      if (response.ok) {
        alert("Form submitted successfully!");
        setEmail("");
        setCrypto("");
      } else {
        alert("Error submitting form.");
      }
    } catch (error) {
      console.error("Error", error);
      alert("Error submitting form");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div className="mb-3">
        <label htmlFor="email" className="form-label">
          Email:
        </label>
        <input
          className="form-control"
          type="email"
          id="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="crypto" className="form-label">
          Cryptocurrency you hold most of:
        </label>
        <input
          className="form-control"
          type="crypto"
          id="crypto"
          value={crypto}
          onChange={(e) => setCrypto(e.target.value)}
          required
        />
      </div>
      <button type="submit" className="btn btn-primary">
        Submit
      </button>
    </form>
  );
};

export default Form;

```

## Integrating the Form into the Application

### Modifying App.tsx

Update App.tsx to include the Form component:

```bash
import React from 'react';
import './App.css';
import Form from './Form';

const App: React.FC = () => {
return (

<div className="App">
<h1>Crypto Holding Form</h1>
<Form />
</div>
);
};

export default App;
```

## Build backend to handle saving of files and uploading them to Apillon's Web3 hosting

Navigate to the parent directory of `my-react-app`

In terminal run:

```bash
mkdir backend
cd backend
npm init -y
npm install express @apillon/sdk cors dotenv
```

### Cretae a quick server with express

In the backend directory we created earlier make new file name server.js and add following content to it:

```bash
const express = require("express");
const fs = require("fs");
const path = require("path");
const cors = require("cors");

const app = express();
const PORT = 5000;

app.use(
  cors({
    origin: "http://localhost:5173",
  })
);

app.use(express.json());

app.post("/form", (req, res) => {
  const { email, crypto } = req.body;

  if (!email || !crypto) {
    res.status(400).json({ error: "Email and cryptocurrency is required!" });
  }

  const fileName = `${email.replace(/[@.]/g, "_")}.json`;
  const filePath = path.join(__dirname, "submissions", fileName);
  const fileContent = `{\n"email": "${email}",\n"crypto": "${crypto}"\n}`;

  fs.writeFile(filePath, fileContent, (err) => {
    if (err) {
      console.error("Error writin file", err);
      return res.status(500).json({ error: "Failed to save submission" });
    }
  });
});

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});

```

Now you have a server that will accept the data sent from your React app in form of json, parse it and save to /submissions/ directory inside of your backend directory.

## Sutup for Apilon Hosting

### Obtaining API Keys

Navigate to [Apillon's](https://apillon.io/) website and create an account. After logging into your dashboard, navigate to ["API Keys"](https://app.apillon.io/dashboard/api-keys), now create and save the keypair data securely.

### Creating a storage bucket

Additionally you need to create a bucket for storing your data in, to do that you need to naviagte to [Storage](https://app.apillon.io/dashboard/service/storage) section and press the "New bucket" button. This will create a bucket with an assigned UUID.

### Configure environment variables

Navigate to the backend directory and create a `.env` file and add your API key, API secret and bucket UUID.

should look something like this(obviously replace values with your actual keys):

```bash
APILLON_BUCKET_UUID=<YOUR BUCKET UUID>
APILLON_API_KEY=<YOUR API KEY>
APILLON_API_SECRET=<YOUR API KEY SECRET>
```

### Create the script to upload the files to your decentralized storage bucket

Create a `fileUpload.js` file in the backend folder with the following code:

```bash
// Importing necessary modules and classes from the @apillon/sdk package
const { Storage, LogLevel } = require("@apillon/sdk");

// Importing the file system module to read files from the local system
const fs = require("fs");

// Loading environment variables from a .env file into process.env
require("dotenv").config();

// Storing API key, secret, and bucket UUID from environment variables
const APILLON_API_KEY = process.env.APILLON_API_KEY;
const APILLON_API_SECRET = process.env.APILLON_API_SECRET;
const APILLON_BUCKET_UUID = process.env.APILLON_BUCKET_UUID;

// Defining an asynchronous function to handle file upload
const handleUpload = async (savedFileName) => {
  // Creating an instance of the Storage class with API credentials and log level
  const storage = new Storage({
    key: APILLON_API_KEY, // API key for authentication
    secret: APILLON_API_SECRET, // API secret for authentication
    logLevel: LogLevel.VERBOSE, // Setting the log level to verbose for detailed logging
  });

  // Getting a reference to the specific bucket using the bucket UUID
  const bucket = storage.bucket(APILLON_BUCKET_UUID);

  // Reading the file to be uploaded from the local filesystem into a buffer
  const savedFileBuffer = fs.readFileSync("./submissions/" + savedFileName);

  // Uploading the file to the specified bucket with additional options
  await bucket.uploadFiles(
    [
      {
        fileName: savedFileName, // Name of the file to be uploaded
        contentType: "application/json", // MIME type of the file
        content: savedFileBuffer, // File content as a buffer
      },
    ],
    { wrapWithDirectory: true, directoryPath: "/ReactFormData" } // Upload options including directory path
  );
};

// Exporting the handleUpload function for use in other modules
module.exports = handleUpload;
```

### Tie the server and the upload script together

Import `handleUpload` to `server.js` by adding at the very beginning of the file.

```bash
const handleUpload = require("./filesUpload");
```

Now that you have the function imported, lets call it from `app.post` and pass the filename of the saved data as a parameter for the SDK to find the desired file in `submissions` dir.

```bash
app.post("/form", (req, res) => {
  const { email, crypto } = req.body;

  if (!email || !crypto) {
    res.status(400).json({ error: "Email and cryptocurrency is required!" });
  }

  const fileName = `${email.replace(/[@.]/g, "_")}.json`;
  const filePath = path.join(__dirname, "submissions", fileName);
  const fileContent = `{\n"email": "${email}",\n"crypto": "${crypto}"\n}`;

  fs.writeFile(filePath, fileContent, (err) => {
    if (err) {
      console.error("Error writin file", err);
      return res.status(500).json({ error: "Failed to save submission" });
    }

    // here is the call to function with the passed fileName variable
    handleUpload(fileName);
    res.status(200).json({ message: "Form submitted successfully" });
    //
  });
});
```

## Running the Application

Now it's time to run both the Server and the Vite development server.

### Starting the Server

Open terminal in the `backend` directory and run:

```bash
npm start
```

#### You should see an output like this:

```bash
> backend@1.0.0 start
> node server.js

Server is running on http://localhost:5000
```

### Starting the Vite Development Server

Open another terminal and run:

```bash
npm run dev
```

## Testing the Application

Open your browser and navigate to http://localhost:5173. Fill out the form and submit it. The data should be sent to the server, saved `submissions` and sent to Web3 storage using Apillon's SDK.

### Troubleshooting

#### Common Issues and Fixes

If you encounter a net::ERR_CONNECTION_REFUSED error, ensure that the JSON Server is running by executing npm run server and checking the output.

#### Network and CORS Issues

By default, JSON Server should handle CORS, but if you encounter issues, you may need to configure CORS in the JSON Server setup.

# Code Repository

You can find the complete source code for this tutorial [here](https://github.com/ravil-who-codes).

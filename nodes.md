# Azure Service Bus POC Guide (.NET + React) - With Code

This guide provides the full code and step-by-step terminal commands to build your Proof of Concept in the `/Users/rajanpawar/snehal/github/azure-tutorials` folder, including setting up your prerequisites and version control.

---

## Prerequisites Installation (Mac)

Since you are on a Mac, you can install the necessary tools using the official installers or Homebrew.

### 1. Install Node.js (For React)
Node.js comes with `npm` (Node Package Manager), which is required to create and run React apps.
- **Option A (Official Installer):** Go to [nodejs.org](https://nodejs.org/), download the **LTS (Long Term Support)** version for macOS, and run the installer.
- **Option B (Homebrew):** If you use Homebrew, open your terminal and run: `brew install node`
- **Verify:** Open your terminal and run `node -v` and `npm -v`. Both should print a version number.

### 2. Install .NET SDK (For Backend)
The .NET SDK allows you to build and run C# applications.
- **Option A (Official Installer):** Go to [dotnet.microsoft.com/download](https://dotnet.microsoft.com/download), download the **.NET 8.0 SDK** (or latest LTS) for macOS, and run the installer.
- **Option B (Homebrew):** Run: `brew install --cask dotnet-sdk`
- **Verify:** Open your terminal and run `dotnet --version`.

---

## Step 0: Set up Git & GitHub Repository

Before writing code, let's initialize a Git repository in your tutorials folder to manage version control.

### 1. Initialize Git Locally
Open your terminal and run:
```bash
cd /Users/rajanpawar/snehal/github/azure-tutorials
git init
```

### 2. Create a `.gitignore` File
You want to avoid committing heavy build files (`node_modules`, `bin`, `obj`). Create a file named `.gitignore` in the `azure-tutorials` folder and add this:
```text
# Node
node_modules/
dist/

# .NET
bin/
obj/
appsettings.Development.json
```
*(Note: It is also best practice to ignore `appsettings.json` in a real project so you don't leak connection strings, but for this local learning POC, just be careful not to make the repository public if it contains real cloud credentials!)*

### 3. Create the GitHub Repository
1. Go to [github.com](https://github.com/) and log in.
2. Click the **"+"** icon in the top right and select **New repository**.
3. Name it `azure-tutorials`.
4. Choose **Private** (recommended, since you will have connection strings in your code) or Public.
5. Do **not** check "Add a README" or "Add .gitignore" (we are doing this locally).
6. Click **Create repository**.

### 4. Link Local to GitHub
On the next page on GitHub, copy the commands under *"…or push an existing repository from the command line"*. It will look like this (run this in your terminal):
```bash
git add .
git commit -m "Initial commit: Set up repository"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/azure-tutorials.git
git push -u origin main
```

---

## Step 1: Set up Azure Resources
1. Go to [portal.azure.com](https://portal.azure.com/).
2. Create a **Service Bus Namespace** (Basic tier is sufficient).
3. Inside your namespace, create a **Queue** named `poc-queue`.
4. Go to **Shared access policies**, click `RootManageSharedAccessKey`, and copy the **Primary Connection String**. Keep this handy.

---

## Step 2: Create the .NET Backend

Open your terminal and make sure you are in your tutorials folder:
```bash
cd /Users/rajanpawar/snehal/github/azure-tutorials
```

### 1. Scaffold the API
```bash
dotnet new webapi -n ServiceBusBackend
cd ServiceBusBackend
dotnet add package Azure.Messaging.ServiceBus
```

### 2. Configure `appsettings.json`
Open `appsettings.json` inside `ServiceBusBackend` and add your Azure Connection String:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ServiceBus": {
    "ConnectionString": "YOUR_COPIED_CONNECTION_STRING_HERE",
    "QueueName": "poc-queue"
  }
}
```

### 3. Create the Message Sender Service
Create a new file called `ServiceBusSenderService.cs` in the root of `ServiceBusBackend`:
```csharp
using Azure.Messaging.ServiceBus;

namespace ServiceBusBackend;

public class ServiceBusSenderService
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;

    public ServiceBusSenderService(IConfiguration configuration)
    {
        var connectionString = configuration["ServiceBus:ConnectionString"];
        var queueName = configuration["ServiceBus:QueueName"];
        
        _client = new ServiceBusClient(connectionString);
        _sender = _client.CreateSender(queueName);
    }

    public async Task SendMessageAsync(string messageBody)
    {
        ServiceBusMessage message = new ServiceBusMessage(messageBody);
        await _sender.SendMessageAsync(message);
    }
}
```

### 4. Create the API Controller
Create a folder named `Controllers` (if it doesn't exist) and add `MessagesController.cs`:
```csharp
using Microsoft.AspNetCore.Mvc;

namespace ServiceBusBackend.Controllers;

[ApiController]
[Route("api/[controller]")]
public class MessagesController : ControllerBase
{
    private readonly ServiceBusSenderService _serviceBusSender;

    public MessagesController(ServiceBusSenderService serviceBusSender)
    {
        _serviceBusSender = serviceBusSender;
    }

    [HttpPost]
    public async Task<IActionResult> SendMessage([FromBody] MessageRequest request)
    {
        if (string.IsNullOrEmpty(request.Text))
            return BadRequest("Message cannot be empty");

        await _serviceBusSender.SendMessageAsync(request.Text);
        
        return Ok(new { status = "Message sent to Azure Service Bus successfully!" });
    }
}

public class MessageRequest
{
    public string Text { get; set; } = string.Empty;
}
```

### 5. Update `Program.cs`
Update your `Program.cs` to register the service, add controllers, and enable CORS so the React app can connect:
```csharp
using ServiceBusBackend;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddSingleton<ServiceBusSenderService>();

// Enable CORS for React Frontend
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowReactApp", policy =>
    {
        policy.WithOrigins("http://localhost:5173") // Vite default port
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

var app = builder.Build();

app.UseCors("AllowReactApp");
app.MapControllers();

app.Run();
```

Test your backend by running: `dotnet run`

---

## Step 3: Create the React Frontend

Open a **new** terminal window, navigate back to the tutorials folder, and create the React app:
```bash
cd /Users/rajanpawar/snehal/github/azure-tutorials
npm create vite@latest frontend -- --template react
cd frontend
npm install
```

### 1. Update `App.jsx`
Replace the contents of `src/App.jsx` with the following code:
```jsx
import { useState } from 'react';
import './App.css';

function App() {
  const [message, setMessage] = useState('');
  const [status, setStatus] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const handleSendMessage = async (e) => {
    e.preventDefault();
    if (!message) return;

    setIsLoading(true);
    setStatus('');

    try {
      // Check your .NET terminal output for the correct port (usually 5000, 5001, or 5119)
      const response = await fetch('http://localhost:5119/api/messages', { 
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ text: message }),
      });

      if (response.ok) {
        setStatus('✅ Message sent to Service Bus!');
        setMessage('');
      } else {
        setStatus('❌ Failed to send message.');
      }
    } catch (error) {
      console.error(error);
      setStatus('❌ Error connecting to API.');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div style={{ padding: '2rem', fontFamily: 'sans-serif', maxWidth: '400px', margin: '0 auto' }}>
      <h2>Azure Service Bus POC</h2>
      
      <form onSubmit={handleSendMessage} style={{ display: 'flex', flexDirection: 'column', gap: '1rem' }}>
        <textarea
          value={message}
          onChange={(e) => setMessage(e.target.value)}
          placeholder="Type a message for the queue..."
          rows={4}
          style={{ padding: '0.5rem', width: '100%' }}
        />
        <button 
          type="submit" 
          disabled={isLoading}
          style={{ padding: '0.75rem', backgroundColor: '#0078D4', color: 'white', border: 'none', cursor: 'pointer' }}
        >
          {isLoading ? 'Sending...' : 'Send Message'}
        </button>
      </form>

      {status && <p style={{ marginTop: '1rem', fontWeight: 'bold' }}>{status}</p>}
    </div>
  );
}

export default App;
```

### 2. Run the React App
```bash
npm run dev
```

---

## Step 4: End-to-End Test

1. Ensure **both** your .NET backend and React frontend are running.
2. Open the React app in your browser (`http://localhost:5173`).
3. Type a message and hit "Send".
4. Go to the Azure Portal -> Your Service Bus Namespace -> **Queues** -> `poc-queue`.
5. Click on **Service Bus Explorer** on the left menu.
6. Click **Peek from start** to view the messages that your .NET API successfully placed into the Azure queue!

---

## Step 5: Save your progress to GitHub
Once you have tested the app and everything is working, open a terminal in the `azure-tutorials` folder and commit your changes:
```bash
git add .
git commit -m "Added .NET backend and React frontend for Service Bus POC"
git push origin main
```

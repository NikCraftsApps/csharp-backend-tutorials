# 🔗 Building a LAN Sync System in C#

In this tutorial, we'll explore how to set up a **basic LAN (Local Area Network) synchronization** system using C# and .NET 6. This allows multiple instances of an application running on different machines in the same network to **exchange data** in real time, **without** requiring an external internet service.

---

## 1. ⚙ Overview

We'll create:
- **A simple server** that listens for connections from clients on the same LAN.
- **A client class** that can connect to the server, send messages, and receive updates.
- **A shared data model** to demonstrate synchronization (e.g., a list of transactions, chat messages, etc.).

**Key Points:**
- **TCP** (Transmission Control Protocol) ensures reliable delivery of data.
- Each client and server runs on the same LAN, identified by IP address + port.
- This tutorial focuses on **core logic**. You can adapt it to your specific app.

---

## 2. 📝 Prerequisites

1. **.NET 6+**  
2. Basic understanding of **TCP** sockets in C#
3. (Optional) Knowledge of **asynchronous programming** (`async/await`)

We will install **no extra packages** for the simplest example, just using `System.Net.Sockets`.

---

## 3. 🏗 Creating the LAN Server

Let's define a **LanServer** class that:
- Listens on a **TCP port** (e.g., 5000).
- Accepts incoming client connections.
- Manages client sockets in a list.

```csharp
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Threading.Tasks;
using System.Text;

namespace App.LanSync
{
    public class LanServer
    {
        private readonly int _port;
        private TcpListener? _listener;
        private readonly List<TcpClient> _clients = new();

        public LanServer(int port)
        {
            _port = port;
        }

        public async Task StartAsync()
        {
            _listener = new TcpListener(IPAddress.Any, _port);
            _listener.Start();

            Console.WriteLine($"[LAN SERVER] Listening on port {_port}...");

            while (true)
            {
                var client = await _listener.AcceptTcpClientAsync();
                Console.WriteLine("[LAN SERVER] Client connected.");

                lock (_clients)
                {
                    _clients.Add(client);
                }

                _ = HandleClientAsync(client);
            }
        }

        public void Stop()
        {
            _listener?.Stop();
            lock (_clients)
            {
                foreach (var c in _clients)
                {
                    c.Close();
                }
                _clients.Clear();
            }
        }

        private async Task HandleClientAsync(TcpClient client)
        {
            using var stream = client.GetStream();
            var buffer = new byte[4096];
            try
            {
                while (true)
                {
                    int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
                    if (bytesRead == 0)
                    {
                        // Client disconnected
                        Console.WriteLine("[LAN SERVER] Client disconnected.");
                        break;
                    }

                    var msg = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                    Console.WriteLine($"[LAN SERVER] Received: {msg}");

                    // Broadcast message to all clients:
                    Broadcast(msg, client);
                }
            }
            finally
            {
                lock (_clients)
                {
                    _clients.Remove(client);
                }
                client.Close();
            }
        }

        private void Broadcast(string message, TcpClient sender)
        {
            var data = Encoding.UTF8.GetBytes(message);
            lock (_clients)
            {
                foreach (var c in _clients)
                {
                    if (c == sender) continue; // optional: skip echo to sender
                    try
                    {
                        c.GetStream().Write(data, 0, data.Length);
                    }
                    catch
                    {
                        // handle or remove
                    }
                }
            }
        }
    }
}
```

1. **StartAsync()** – starts `TcpListener` on the specified port and accepts clients in a loop.  
2. **HandleClientAsync()** – reads data from a specific client, then broadcasts to others via `Broadcast()`.  
3. **Broadcast(...)** – sends the received message to all connected clients (except the sender, if desired).

---

## 4. 💻 Creating the LAN Client

A **LanClient** class:
- Connects to the server's IP and port.
- Receives messages in a loop and displays/logs them.
- Allows you to send a message to the server.

```csharp
using System;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;

namespace App.LanSync
{
    public class LanClient
    {
        private TcpClient _tcpClient;
        private NetworkStream _stream;

        public bool IsConnected => _tcpClient.Connected;

        public LanClient()
        {
            _tcpClient = new TcpClient();
        }

        public async Task ConnectAsync(string serverIp, int port)
        {
            await _tcpClient.ConnectAsync(serverIp, port);
            _stream = _tcpClient.GetStream();

            // Start reading in background
            _ = ReadLoopAsync();
        }

        public async Task SendAsync(string message)
        {
            if (!IsConnected) return;
            var data = Encoding.UTF8.GetBytes(message);
            await _stream.WriteAsync(data, 0, data.Length);
        }

        private async Task ReadLoopAsync()
        {
            var buffer = new byte[4096];
            try
            {
                while (true)
                {
                    int bytesRead = await _stream.ReadAsync(buffer, 0, buffer.Length);
                    if (bytesRead == 0)
                    {
                        Console.WriteLine("[LAN CLIENT] Server disconnected.");
                        break;
                    }

                    var msg = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                    Console.WriteLine($"[LAN CLIENT] Received: {msg}");
                    // In a real app, parse the message or update local data
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[LAN CLIENT] Error: {ex.Message}");
            }
        }

        public void Disconnect()
        {
            _tcpClient.Close();
        }
    }
}
```

1. **ConnectAsync** – connects to the server (`serverIp`, `port`), starts `ReadLoopAsync`.  
2. **SendAsync** – sends a message to the server.  
3. **ReadLoopAsync** – continuously receives and logs messages from the server.

---

## 5. 🏁 Testing Locally (Console Demo)

```csharp
// Program.cs (Console test)
using System;
using System.Threading.Tasks;

namespace App.LanSync.ConsoleDemo
{
    class Program
    {
        static async Task Main(string[] args)
        {
            Console.WriteLine("1) Start server");
            Console.WriteLine("2) Start client");
            var choice = Console.ReadLine();

            if (choice == "1")
            {
                var server = new LanServer(5000);
                await server.StartAsync(); // never returns in this simple example
            }
            else if (choice == "2")
            {
                var client = new LanClient();
                await client.ConnectAsync("127.0.0.1", 5000);

                Console.WriteLine("Connected to server. Type messages to send:");
                while (true)
                {
                    var msg = Console.ReadLine();
                    if (string.IsNullOrEmpty(msg)) break;
                    await client.SendAsync(msg);
                }
                client.Disconnect();
            }
        }
    }
}
```

1. **Start server** (option `1`): server listens on port `5000`.  
2. **Start client** (option `2`): client connects to `127.0.0.1:5000`.  
3. Any message typed in the client console is **broadcast** by the server to other clients.

---

## 6. 🔧 Next Steps

1. **Data Synchronization** – parse JSON messages (e.g., a list of transactions), then update local data structures.  
2. **Multiple Clients** – each client sees real-time changes.  
3. **Error Handling & Reconnect** – handle abrupt disconnects, reconnections.  
4. **Security** – optional encryption or authentication if needed.

---

## 7. 📌 Summary

By following this guide, you have a **basic LAN Sync System** in C#:
- **LanServer** – listens for TCP clients, broadcasts messages.
- **LanClient** – connects to the server, sends data, receives updates in real-time.
- You can extend it to synchronize domain data (e.g., MyFinBoard transactions).

Happy coding! If you need advanced features, consider **SignalR** or **UDP** for faster messaging, or integrate a **database** for multi-user sync. For a local network, this simple approach is enough to get started.

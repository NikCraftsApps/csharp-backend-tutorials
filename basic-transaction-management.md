# ✨ Basic Transaction Management in C# ✨

This tutorial demonstrates how to create a **basic CRUD (Create, Read, Update, Delete)** system for managing financial transactions in C#. Although this example focuses on **transaction data**, you can easily adapt it to other scenarios.

---

## 1. 🏗 Overview

We'll build a **TransactionService** that handles:
1. **Add** a new transaction  
2. **Retrieve** (read) all transactions  
3. **Edit** an existing transaction  
4. **Delete** a transaction  

Additionally, we'll show how to **persist** data to a **JSON file** so that changes are saved across application restarts.

---

## 2. ⚙ Prerequisites

- **.NET 6+**  
- Basic knowledge of **C#**  
- Familiarity with **JSON** serialization (we'll use `Newtonsoft.Json`)

Make sure to install:

```bash
dotnet add package Newtonsoft.Json
```

---

## 3. 📝 Defining the Transaction Model

Create a **Transaction** class with the necessary fields. For example:

```csharp
using System;

namespace MyFinBoard.Models
{
    public enum TransactionType
    {
        Income,
        Expense
    }

    public class Transaction
    {
        public Guid Id { get; set; } = Guid.NewGuid();
        public string Name { get; set; } = "";
        public decimal Amount { get; set; }
        public DateTime Date { get; set; } = DateTime.Now;
        public TransactionType Type { get; set; }
        public string Category { get; set; } = "";
        public string Description { get; set; } = "";
        public string Currency { get; set; } = "PLN";
    }
}
```

- **Id**: unique identifier for each transaction  
- **Amount**, **Date**, **Type**: essential financial data  
- **Category**, **Description**, **Currency**: additional fields for more context  

---

## 4. 🗃 Creating the TransactionService

### 4.1. In-Memory List

We'll keep transactions in a **private List**. For persistence, we'll **read and write** from a JSON file each time we modify data.

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using MyFinBoard.Models;
using Newtonsoft.Json;

namespace MyFinBoard.Services
{
    public class TransactionService
    {
        private readonly string _filePath;
        private List<Transaction> _transactions;

        public TransactionService(string filePath = "transactions.json")
        {
            _filePath = filePath;
            _transactions = new List<Transaction>();

            LoadFromFile();
        }

        private void LoadFromFile()
        {
            if (!File.Exists(_filePath))
            {
                _transactions = new List<Transaction>();
                return;
            }

            var json = File.ReadAllText(_filePath);
            _transactions = JsonConvert.DeserializeObject<List<Transaction>>(json) 
                            ?? new List<Transaction>();
        }

        private void SaveToFile()
        {
            var json = JsonConvert.SerializeObject(_transactions, Formatting.Indented);
            File.WriteAllText(_filePath, json);
        }

        // CREATE
        public void AddTransaction(Transaction transaction)
        {
            _transactions.Add(transaction);
            SaveToFile();
        }

        // READ (all)
        public IEnumerable<Transaction> GetAll()
        {
            return _transactions;
        }

        // READ (by id)
        public Transaction? GetById(Guid id)
        {
            return _transactions.FirstOrDefault(t => t.Id == id);
        }

        // UPDATE
        public bool UpdateTransaction(Guid id, Transaction updated)
        {
            var existing = GetById(id);
            if (existing == null) return false;

            // Copy fields
            existing.Name = updated.Name;
            existing.Amount = updated.Amount;
            existing.Date = updated.Date;
            existing.Type = updated.Type;
            existing.Category = updated.Category;
            existing.Description = updated.Description;
            existing.Currency = updated.Currency;

            SaveToFile();
            return true;
        }

        // DELETE
        public bool DeleteTransaction(Guid id)
        {
            var count = _transactions.RemoveAll(t => t.Id == id);
            if (count > 0)
            {
                SaveToFile();
                return true;
            }
            return false;
        }
    }
}
```

1. **LoadFromFile()** – reads transactions from `transactions.json` if it exists.  
2. **SaveToFile()** – writes in-memory list to JSON after every change.  
3. **AddTransaction()** – adds a new transaction, then saves.  
4. **UpdateTransaction()** – modifies an existing transaction’s fields, then saves.  
5. **DeleteTransaction()** – removes one or more matching transactions, then saves.

---

## 5. 💻 Using the TransactionService

### 5.1. Simple Console Demo (Optional)

Below is a quick example showing how you might **test** the service in a console app:

```csharp
using System;
using MyFinBoard.Models;
using MyFinBoard.Services;

namespace MyFinBoard.ConsoleDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            var service = new TransactionService("transactions.json");

            // CREATE
            var newTx = new Transaction
            {
                Name = "Freelance Payment",
                Amount = 500m,
                Date = DateTime.Now,
                Type = TransactionType.Income,
                Category = "Work",
                Description = "Website design project",
                Currency = "USD"
            };
            service.AddTransaction(newTx);

            // READ
            foreach (var tx in service.GetAll())
            {
                Console.WriteLine($"{tx.Id} | {tx.Name} | {tx.Amount} {tx.Currency}");
            }

            // UPDATE
            var updatedTx = new Transaction
            {
                Name = "Freelance Payment (updated)",
                Amount = 600m,
                Date = DateTime.Now,
                Type = TransactionType.Income,
                Category = "Work",
                Description = "Revised invoice",
                Currency = "USD"
            };
            service.UpdateTransaction(newTx.Id, updatedTx);

            // DELETE (optional)
            // service.DeleteTransaction(newTx.Id);

            Console.WriteLine("Done. Press any key to exit.");
            Console.ReadKey();
        }
    }
}
```

- **AddTransaction** – creates a new entry.
- **GetAll** – lists them in console.
- **UpdateTransaction** – modifies an existing transaction.
- **DeleteTransaction** – (optional) removes the transaction.

---

## 6. 🎯 Potential Enhancements

1. **Validation** – ensure `Amount >= 0`, non-empty `Name`, etc.  
2. **Advanced Filtering** – date ranges, categories, transaction types.  
3. **Encryption** – if storing sensitive data, consider encrypting JSON.  
4. **Multi-user** – integrate a database or a syncing mechanism for multiple users.  

---

## 7. 📌 Summary

By following this guide, you now have a **basic transaction system** in C# with:

- **CRUD operations** (Create, Read, Update, Delete)
- **JSON persistence** for data across restarts
- **Optional** advanced scenarios (validation, encryption, multi-user)

Feel free to **connect** this service with a GUI (Avalonia UI) or a web API (ASP.NET Core) to make a fully functional financial application.

# Real-Time-Notification-Handling-with-PostgreSQL-and-Python
This setup allows you to receive real-time notifications from PostgreSQL and call an open API based on these notifications. The process is adaptable for both standalone scripts and Jupyter Notebooks, ensuring flexibility in how you run and test your listener.

## Setup Instructions

### 1. Set Up PostgreSQL Table and Trigger

#### 1.1 Create the PostgreSQL Table

Execute the following SQL commands in your PostgreSQL client to create a table that will be monitored for changes:

```sql
-- Create the table
CREATE TABLE people (
    pk serial PRIMARY KEY,
    ts TIMESTAMPTZ DEFAULT now(),
    name TEXT,
    age INTEGER,
    height REAL
);
```

#### 1.2 Create the Trigger Function and Trigger
Create a trigger function and trigger to send notifications when changes occur in the table:

```sql
-- Create the trigger function
CREATE OR REPLACE FUNCTION notify_data_changes() RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('data_changes', json_build_object(
        'schema_name', TG_TABLE_SCHEMA,
        'table_name', TG_TABLE_NAME,
        'action', TG_OP,
        'data', COALESCE(ROW(NEW.*)::TEXT, ROW(OLD.*)::TEXT)
    )::TEXT);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;


---Create the trigger
CREATE TRIGGER people_changes
AFTER INSERT OR UPDATE OR DELETE ON people
FOR EACH ROW EXECUTE FUNCTION notify_data_changes();
```

### 2. Set Up the Python Listener Script
#### 2.1 Install Dependencies
Ensure you have the required Python libraries installed. Run the following command:

```bash
pip install asyncpg aiohttp
```

#### 2.2 Create the Python Script
Create a file named listener.py with the following content:

```python
import asyncio
import asyncpg
import aiohttp

# Function to handle notifications
async def notification_handler(connection, pid, channel, payload):
    print(f"Received notification: {payload}")

    # URL of the open API
    api_url = 'https://jsonplaceholder.typicode.com/posts' ###Sample API for testing
    
    async with aiohttp.ClientSession() as session:
        # Call the API and print the response
        async with session.post(api_url, json={'data': payload}) as resp:
            print(f"API response status: {resp.status}")
            json_response = await resp.json()
            print(f"API response body: {json_response}")

# Function to handle notifications continuously
async def handle_notifications():
    # Connect to the PostgreSQL database
    conn = await asyncpg.connect(dsn='connection url')
    # Add a listener for the 'data_changes' channel
    await conn.add_listener('data_changes', notification_handler)
    try:
        print('Listening for notifications...')
        await asyncio.sleep(3600)  # Keep the listener running for 1 hour
    finally:
        await conn.close()

# Main function to start the notification handler
async def main():
    await handle_notifications()

# Run the main function
if __name__ == "__main__":
    asyncio.run(main())
```

#### 2.3 Run the Python Script
Save the script and execute it with:
```python
python listener.py
```

### 3. Run the Listener in Jupyter Notebook
To run the listener script within a Jupyter Notebook, adapt the script to work with the notebook’s event loop.

#### 3.1 Install Dependencies
Ensure the required libraries are installed in your Jupyter Notebook environment:

```python
!pip install asyncpg aiohttp
```

#### 3.2 Adapted Listener Code for Jupyter Notebook
Create a new Jupyter Notebook cell with the following code:

```python
import asyncio
import asyncpg
import aiohttp
import nest_asyncio

# Allow nested event loops in Jupyter Notebook
nest_asyncio.apply()

# Function to handle notifications
async def notification_handler(connection, pid, channel, payload):
    print(f"Received notification: {payload}")

    # URL of the open API
    api_url = 'https://jsonplaceholder.typicode.com/posts'
    
    async with aiohttp.ClientSession() as session:
        # Call the API and print the response
        async with session.post(api_url, json={'data': payload}) as resp:
            print(f"API response status: {resp.status}")
            json_response = await resp.json()
            print(f"API response body: {json_response}")

# Function to handle notifications continuously
async def handle_notifications():
    # Connect to the PostgreSQL database
    conn = await asyncpg.connect(dsn='connection url')
    # Add a listener for the 'data_changes' channel
    await conn.add_listener('data_changes', notification_handler)
    try:
        print('Listening for notifications...')
        await asyncio.sleep(3600)  # Keep the listener running for 1 hour
    finally:
        await conn.close()

# Main function to start the notification handler
async def main():
    await handle_notifications()

# Run the main function
await main()
```
### Explanation

#### PostgreSQL Setup
Trigger Function: The notify_data_changes function sends a notification using pg_notify when data changes in the people table.
Trigger: The people_changes trigger activates the notify_data_changes function after INSERT, UPDATE, or DELETE operations.

#### Python Listener
Notification Handler: The notification_handler function processes notifications and calls the JSONPlaceholder API with the notification payload.
Handle Notifications: The handle_notifications function connects to PostgreSQL, listens for notifications, and keeps running.

#### Jupyter Notebook Integration
Async Handling: The notebook script uses nest_asyncio to handle asynchronous operations within Jupyter’s event loop. It runs the notification handler directly in the notebook.

#### References:
https://www.crunchydata.com/blog/real-time-database-events-with-pg_eventserv 

https://www.cybertec-postgresql.com/en/listen-notify-automatic-client-notification-in-postgresql/ 

https://github.com/crunchydata/pg_eventserv?tab=readme-ov-file



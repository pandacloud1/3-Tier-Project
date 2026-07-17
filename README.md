# 3-Tier-Project

🛠️ Step 1: Install Python 3.8 and Packages (App Tier EC2)
```sh
sudo apt-get install software-properties-common -y
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt-get update
sudo apt-get install python3.8 python3.8-venv python3.8-distutils -y
```

Install pip for Python 3.8
```sh
curl -sS https://bootstrap.pypa.io/get-pip.py | sudo python3.8
```

Create virtual environment
```py
python3.8 -m venv venv
source venv/bin/activate
```

Install required packages
```sh
pip install Flask mysql-connector-python flask-cors
```

🛠️ Step 2: Create Database and Table (RDS)
Log into MySQL:

```bash
mysql -h <RDS-ENDPOINT> -u admin -p
```
Run:

```sql
CREATE DATABASE todo_db;
USE todo_db;

CREATE TABLE tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task VARCHAR(255) NOT NULL
);
```

🛠️ Step 3: Backend Code (main.py on App Tier EC2)
```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import mysql.connector

app = Flask(__name__)
CORS(app)

db = mysql.connector.connect(
    host="your-rds-endpoint.rds.amazonaws.com",
    user="admin",
    password="yourpassword",
    database="todo_db"
)

cursor = db.cursor()

@app.route('/add', methods=['POST'])
def add_task():
    data = request.get_json()
    task = data.get('task')
    cursor.execute("INSERT INTO tasks (task) VALUES (%s)", (task,))
    db.commit()
    return jsonify({"message": "Task added!"})

@app.route('/list', methods=['GET'])
def list_tasks():
    cursor.execute("SELECT * FROM tasks")
    tasks = cursor.fetchall()
    return jsonify(tasks)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Run it:

```bash
source venv/bin/activate
python3.8 main.py
```

🛠️ Step 4: Install Apache2 (Web Tier EC2)
```bash
sudo apt update
sudo apt install apache2 -y
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo systemctl restart apache2
```

Edit Apache config:
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```
Add inside <VirtualHost *:80>:

```apache
ProxyPass "/api" "http://<APP-TIER-PRIVATE-IP>:5000/"
ProxyPassReverse "/api" "http://<APP-TIER-PRIVATE-IP>:5000/"
```

Restart Apache:
```bash
sudo systemctl restart apache2
```

🛠️ Step 5: Frontend Code (index.html on Web Tier EC2)
Save to /var/www/html/index.html:
```html
<!DOCTYPE html>
<html>
<head>
  <title>To-Do List</title>
</head>
<body>
  <h1>My To-Do List</h1>
  <input type="text" id="taskInput" placeholder="Enter task">
  <button onclick="addTask()">Add Task</button>
  <button onclick="loadTasks()">List Tasks</button>
  <ul id="taskList"></ul>

  <script>
    async function addTask() {
      const task = document.getElementById("taskInput").value;
      await fetch("/api/add", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ task })
      });
      document.getElementById("taskInput").value = "";
      loadTasks();
    }

    async function loadTasks() {
      const response = await fetch("/api/list");
      const tasks = await response.json();
      const list = document.getElementById("taskList");
      list.innerHTML = "";
      tasks.forEach(t => {
        list.innerHTML += `<li>${t[1]}</li>`;
      });
    }
  </script>
</body>
</html>
```

🛠️ Step 6: Verify End‑to‑End
Open browser → `http://<WEB-TIER-PUBLIC-IP>/index.html`

Add a task → Web → App → RDS.

Click List Tasks → tasks appear from DB.

🛠️ Step 7: Verify from RDS DB
```bash
mysql -h <rds-endpoint> -u admin -p
```

```mysql
USE todo_db;
```

```mysql
SELECT * FROM tasks;
```

✨ With this, you have a complete 3‑tier app demo:
- Web Tier (Apache) serves frontend and proxies /api → App Tier.
- App Tier (Flask) handles API requests.
- DB Tier (RDS) stores tasks.

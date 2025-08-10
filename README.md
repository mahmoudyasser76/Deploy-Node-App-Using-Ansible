# Flask App Deployment on AWS EC2

Deploy a Note-Taking Website on AWS EC2 with Backup Strategy using ansible role.

---

## 📂 Project Structure

```
noteapp/
│
├── ansible.cfg
├── hosts
├── playbook.yml
│
├── roles/
│   ├── setup/
│   │   └── tasks/main.yml
│   ├── deploy/
│   │   ├── files/
│   │   │   ├── app.py
│   │   │   ├── templates/
│   │   │   │   └── index.html
│   │   │   ├── static/
│   │   │   │   ├── style.css
│   │   │   ├── requirements.txt
│   │   │   └── backup.sh
│   │   └── tasks/main.yml

```

---

## 🚀 Deployment Steps

1. **Install Ansible**: Ensure Ansible is installed on your local machine.
   ```bash
   sudo dnf install ansible
   ```
2. **Configure Inventory**: Update the `hosts` file with your EC2 instance details.
   ```ini
    [web]
    98.81.159.0 ansible_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/noteapp/noteapp.pem ansible_python_interpreter=/usr/bin/python3.9
   ```
3. **Run the Playbook**: Execute the Ansible playbook to set up the environment and deploy the application.
   ```bash
   ansible-playbook -i hosts playbook.yml
   ```
4. **Access the Application**: Open your web browser and navigate to `http://98.81.159.0` to view the Note-Taking Website.

---

## 🛠️ Ansible Role Breakdown

    ```bash
    noteapp/
    ├── ansible.cfg
    ├── hosts
    ├── playbook.yml
    ├── roles/
    │   ├── setup/
    │   │   └── tasks/main.yml
    │   ├── deploy/
    │   │   ├── files/
    │   │   │   ├── app.py
    │   │   │   ├── templates/
    │   │   │   │   └── index.html
    │   │   │   ├── static/
    │   │   │   │   ├── style.css
    │   │   │   ├── requirements.txt
    │   │   │   └── backup.sh
    │   │   └── tasks/main.yml
    ```

## ansible.cfg

```ini
[defaults]
inventory = ./hosts
host_key_checking = False
#private_key_file = ./ansible.pem
#warn_missing_interpreter = False
```

## hosts

```ini
[web]
98.81.159.0 ansible_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/noteapp/noteapp.pem ansible_python_interpreter=/usr/bin/python3.9
```

## playbook.yml

```yaml
---
- name: Deploy Flask App
- hosts: web
  become: yes
  roles:
    - setup
    - deploy
```

## roles/setup/tasks/main.yml

```yaml
---
- name: Update system packages
  yum:
    name: "*"
    state: latest

- name: Install Python3 and pip
  yum:
    name:
      - python3
      - python3-pip
      - sqlite
    state: present
```

## roles/deploy/tasks/main.yml

```yaml
---
- name: Copy application files
  copy:
    src: app.py
    dest: /home/ec2-user/app.py
    mode: "0644"

- name: Copy templates folder
  copy:
    src: templates/
    dest: /home/ec2-user/templates/
    mode: "0644"

- name: Copy static folder
  copy:
    src: static/
    dest: /home/ec2-user/static/
    mode: "0644"

- name: Copy requirements.txt
  copy:
    src: requirements.txt
    dest: /home/ec2-user/requirements.txt
    mode: "0644"

- name: Copy backup script
  copy:
    src: backup.sh
    dest: /home/ec2-user/backup.sh
    mode: "0755"

- name: Install dependencies
  pip:
    requirements: /home/ec2-user/requirements.txt
    executable: pip3

- name: Initialize SQLite database
  shell: |
    python3 - <<EOF
    import sqlite3
    conn = sqlite3.connect('/home/ec2-user/notes.db')
    c = conn.cursor()
    c.execute("""
    CREATE TABLE IF NOT EXISTS notes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        content TEXT,
        created_at TEXT
    )
    """)
    conn.commit()
    conn.close()
    EOF

- name: Run Flask app in background
  shell: |
    nohup python3 /home/ec2-user/app.py > app.log 2>&1 &
```

# roles/deploy/files/app.py

```python
from flask import Flask, request, render_template
import sqlite3
from datetime import datetime
import os

app = Flask(__name__)

DB_PATH = "/home/ec2-user/notes.db"

def get_db_connection():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        content = request.form["content"]
        created_at = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        conn = get_db_connection()
        conn.execute("INSERT INTO notes (content, created_at) VALUES (?, ?)", (content, created_at))
        conn.commit()
        conn.close()
    conn = get_db_connection()
    notes = conn.execute("SELECT * FROM notes ORDER BY id DESC").fetchall()
    conn.close()
    return render_template("index.html", notes=notes)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

# roles/deploy/files/templates/index.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>My Notes</title>
    <link rel="stylesheet" href="/static/style.css" />
  </head>
  <body>
    <h1>My Notes</h1>

    <form method="post">
      <textarea
        name="content"
        placeholder="Write your note here..."
        required
      ></textarea>
      <button type="submit">Save Note</button>
    </form>

    {% for note in notes %}
    <div class="note">
      <time> {{ note['created_at'] }}</time>
      <p>{{ note['content'] }}</p>
    </div>
    {% endfor %}
  </body>
</html>
```

# roles/deploy/files/static/style.css

```css
body {
  font-family: Arial, sans-serif;
  background-color: #636161;
  padding: 30px;
  max-width: 700px;
  margin: auto;
}

h1 {
  color: #ffffff;
  text-align: center;
}

form {
  margin-bottom: 30px;
  display: flex;
  flex-direction: column;
}

textarea {
  min-height: 100px;
  padding: 10px;
  font-size: 16px;
  border-radius: 5px;
  border: 1px solid #ccc;
  margin-bottom: 15px;
}

button {
  background-color: #4caf50;
  color: white;
  padding: 12px;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
}

.note {
  background-color: #fff;
  padding: 15px;
  margin-bottom: 15px;
  border-left: 4px solid #4caf50;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.note time {
  font-size: 0.85em;
  color: #888;
  display: block;
  margin-bottom: 5px;
}

.note p {
  margin: 0;
}

.note button {
  background-color: #f44336;
  margin-top: 10px;
}
```

---

# roles/deploy/files/backup.sh

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
cp /home/ec2-user/notes.db /home/ec2-user/backup_notes_$DATE.db
echo "Backup created: backup_notes_$DATE.db"
```

# roles/deploy/files/requirements.txt

```
Flask==2.0.1
```

---

# Application Screenshot

## ![My Notes](images/My%20notes.png)

---

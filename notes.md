# REST API Design and Implementation Guide

This document provides a comprehensive guide to designing and implementing REST APIs, including:

1. **REST API Methods and Usage**
2. **Python Flask Implementation**
3. **Authentication and Authorization**
4. **HTTP Status Codes for API Design**

---

## 1. REST API Methods and Usage

REST APIs use HTTP methods to perform operations on resources. Here are the most common methods and when to use them:

### **GET**

- **Purpose**: Retrieve data from the server.
- **When to use**:
  - Fetching a list of resources (e.g., `/users`).
  - Fetching a specific resource by ID (e.g., `/users/123`).
  - The request should not modify any data on the server.
- **Example**: Fetch details of a user with ID 123.
  ```
  GET /users/123
  ```

### **POST**

- **Purpose**: Create a new resource or submit data to the server.
- **When to use**:
  - Creating a new resource (e.g., adding a new user).
  - Submitting data to be processed (e.g., form data).
  - Triggering actions that do not fit into other methods.
- **Example**: Create a new user.
  ```
  POST /users
  Body: { "name": "John", "email": "john@example.com" }
  ```

### **PUT**

- **Purpose**: Update an existing resource or create it if it doesn't exist.
- **When to use**:
  - Updating an entire resource (replace all fields).
  - Creating a resource if it doesn't exist (idempotent).
- **Example**: Update the details of a user with ID 123.
  ```
  PUT /users/123
  Body: { "name": "John Doe", "email": "johndoe@example.com" }
  ```

### **PATCH**

- **Purpose**: Partially update an existing resource.
- **When to use**:
  - Updating specific fields of a resource without replacing the entire resource.
- **Example**: Update only the email of a user with ID 123.
  ```
  PATCH /users/123
  Body: { "email": "newemail@example.com" }
  ```

### **DELETE**

- **Purpose**: Delete a resource.
- **When to use**:
  - Removing a specific resource by ID.
- **Example**: Delete a user with ID 123.
  ```
  DELETE /users/123
  ```

### **HEAD**

- **Purpose**: Retrieve only the headers of a response (no body).
- **When to use**:
  - Checking if a resource exists.
  - Fetching metadata (e.g., content type, content length) without downloading the entire resource.
- **Example**: Check if a user with ID 123 exists.
  ```
  HEAD /users/123
  ```

### **OPTIONS**

- **Purpose**: Retrieve the supported HTTP methods for a resource.
- **When to use**:
  - Discovering what operations are allowed on a resource (e.g., CORS preflight requests).
- **Example**: Check allowed methods for the `/users` endpoint.
  ```
  OPTIONS /users
  ```

---

## 2. Python Flask Implementation

Here’s an example implementation of a REST API using **Python Flask**:

```python
from flask import Flask, jsonify, request, abort

app = Flask(__name__)

# In-memory database (for demonstration purposes)
users = [
    {"id": 1, "name": "John Doe", "email": "john@example.com"},
    {"id": 2, "name": "Jane Smith", "email": "jane@example.com"},
]

# Helper function to find a user by ID
def find_user(user_id):
    return next((user for user in users if user["id"] == user_id), None)

# GET all users
@app.route('/users', methods=['GET'])
def get_users():
    return jsonify(users)

# GET a specific user by ID
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = find_user(user_id)
    if user is None:
        abort(404, description="User not found")
    return jsonify(user)

# POST - Create a new user
@app.route('/users', methods=['POST'])
def create_user():
    if not request.json or not 'name' in request.json or not 'email' in request.json:
        abort(400, description="Invalid request: Name and email are required")

    new_user = {
        "id": len(users) + 1,
        "name": request.json['name'],
        "email": request.json['email'],
    }
    users.append(new_user)
    return jsonify(new_user), 201

# PUT - Update an entire user (replace all fields)
@app.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    user = find_user(user_id)
    if user is None:
        abort(404, description="User not found")

    if not request.json:
        abort(400, description="Invalid request: JSON body is required")

    user["name"] = request.json.get('name', user["name"])
    user["email"] = request.json.get('email', user["email"])
    return jsonify(user)

# PATCH - Partially update a user (update specific fields)
@app.route('/users/<int:user_id>', methods=['PATCH'])
def patch_user(user_id):
    user = find_user(user_id)
    if user is None:
        abort(404, description="User not found")

    if not request.json:
        abort(400, description="Invalid request: JSON body is required")

    if 'name' in request.json:
        user["name"] = request.json['name']
    if 'email' in request.json:
        user["email"] = request.json['email']
    return jsonify(user)

# DELETE - Delete a user
@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    user = find_user(user_id)
    if user is None:
        abort(404, description="User not found")

    users.remove(user)
    return jsonify({"message": "User deleted"}), 200

# Error handler for 404
@app.errorhandler(404)
def not_found(error):
    return jsonify({"error": str(error)}), 404

# Error handler for 400
@app.errorhandler(400)
def bad_request(error):
    return jsonify({"error": str(error)}), 400

# Run the Flask app
if __name__ == '__main__':
    app.run(debug=True)
```

---

## 3. Authentication and Authorization

### **Authentication vs Authorization**

1. **Authentication**:

   - Verifies the identity of a user (e.g., login with username/password or tokens).
   - Example: A user provides credentials to prove they are who they claim to be.

2. **Authorization**:
   - Determines what actions an authenticated user is allowed to perform.
   - Example: A user with "admin" role can delete resources, while a "user" role can only read them.

### **Implementation in Flask**

Here’s an example implementation of **JWT-based authentication** and **role-based authorization**:

```python
from flask import Flask, jsonify, request, abort
from flask_jwt_extended import (
    JWTManager, create_access_token, jwt_required, get_jwt_identity
)

app = Flask(__name__)
app.config['JWT_SECRET_KEY'] = 'your-secret-key-here'  # Change in production!
jwt = JWTManager(app)

# In-memory "databases" (replace with real databases in production)
users = [
    {"id": 1, "username": "admin", "password": "admin123", "role": "admin"},
    {"id": 2, "username": "user1", "password": "user123", "role": "user"}
]

notes = [
    {"id": 1, "user_id": 2, "content": "My first note"},
    {"id": 2, "user_id": 2, "content": "Another note"}
]

# Helper functions
def find_user(username):
    return next((u for u in users if u["username"] == username), None)

def find_note(note_id):
    return next((n for n in notes if n["id"] == note_id), None)

# Authentication: Login to get JWT token
@app.route('/login', methods=['POST'])
def login():
    if not request.json or 'username' not in request.json or 'password' not in request.json:
        abort(400, description="Username and password required")

    user = find_user(request.json['username'])
    if not user or user['password'] != request.json['password']:
        abort(401, description="Invalid credentials")

    access_token = create_access_token(identity={
        "username": user["username"],
        "role": user["role"],
        "user_id": user["id"]
    })
    return jsonify(access_token=access_token)

# Authorization: Admin role required decorator
def admin_required(fn):
    @jwt_required()
    def wrapper(*args, **kwargs):
        current_user = get_jwt_identity()
        if current_user["role"] != "admin":
            abort(403, description="Admin access required")
        return fn(*args, **kwargs)
    return wrapper

# Get all notes (authenticated users only)
@app.route('/notes', methods=['GET'])
@jwt_required()
def get_notes():
    current_user = get_jwt_identity()
    user_notes = [n for n in notes if n["user_id"] == current_user["user_id"]]
    return jsonify(user_notes)

# Create new note (authenticated users)
@app.route('/notes', methods=['POST'])
@jwt_required()
def create_note():
    current_user = get_jwt_identity()

    if not request.json or 'content' not in request.json:
        abort(400, description="Note content required")

    new_note = {
        "id": len(notes) + 1,
        "user_id": current_user["user_id"],
        "content": request.json['content']
    }
    notes.append(new_note)
    return jsonify(new_note), 201

# Delete note (note owner or admin)
@app.route('/notes/<int:note_id>', methods=['DELETE'])
@jwt_required()
def delete_note(note_id):
    current_user = get_jwt_identity()
    note = find_note(note_id)

    if not note:
        abort(404, description="Note not found")

    if note["user_id"] != current_user["user_id"] and current_user["role"] != "admin":
        abort(403, description="Not authorized to delete this note")

    notes.remove(note)
    return jsonify({"message": "Note deleted"})

# Admin-only endpoint: Delete user
@app.route('/users/<int:user_id>', methods=['DELETE'])
@admin_required
def delete_user(user_id):
    user = next((u for u in users if u["id"] == user_id), None)
    if not user:
        abort(404, description="User not found")

    users.remove(user)
    return jsonify({"message": "User deleted"})

# Error handlers
@app.errorhandler(400)
def bad_request(error):
    return jsonify({"error": str(error)}), 400

@app.errorhandler(401)
def unauthorized(error):
    return jsonify({"error": str(error)}), 401

@app.errorhandler(403)
def forbidden(error):
    return jsonify({"error": str(error)}), 403

@app.errorhandler(404)
def not_found(error):
    return jsonify({"error": str(error)}), 404

if __name__ == '__main__':
    app.run(debug=True)
```

---

## 4. HTTP Status Codes for API Design

### **1xx: Informational**

- **100 Continue**: The server has received the request headers, and the client should proceed to send the request body.
- **101 Switching Protocols**: The server is switching protocols as requested by the client.

### **2xx: Success**

- **200 OK**: The request was successful.
- **201 Created**: The request was successful, and a new resource was created.
- **204 No Content**: The request was successful, but there is no content to send.

### **3xx: Redirection**

- **301 Moved Permanently**: The requested resource has been permanently moved to a new URL.
- **302 Found**: The requested resource has been temporarily moved to a different URL.
- **304 Not Modified**: The resource has not been modified since the last request.

### **4xx: Client Errors**

- **400 Bad Request**: The request was malformed or invalid.
- **401 Unauthorized**: The client must authenticate itself to access the resource.
- **403 Forbidden**: The client does not have permission to access the resource.
- **404 Not Found**: The requested resource does not exist.
- **405 Method Not Allowed**: The HTTP method is not supported for the requested resource.
- **409 Conflict**: The request conflicts with the current state of the server.
- **422 Unprocessable Entity**: The request was well-formed but contains semantic errors.

### **5xx: Server Errors**

- **500 Internal Server Error**: A generic error message indicating an unexpected server failure.
- **501 Not Implemented**: The server does not support the functionality required to fulfill the request.
- **502 Bad Gateway**: The server, while acting as a gateway or proxy, received an invalid response from an upstream server.
- **503 Service Unavailable**: The server is temporarily unavailable.
- **504 Gateway Timeout**: The server, while acting as a gateway or proxy, did not receive a timely response from an upstream server.

---

This document provides a complete guide to REST API design, implementation, and best practices. Use it as a reference for building robust and secure APIs.

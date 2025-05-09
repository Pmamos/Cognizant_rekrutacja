# Task Management API – Full Stack Exercise

## 1.	Asynchronous Notification

To improve the performance of the application and avoid blocking the main server thread, the `update_task` route was refactored.

Instead of executing the notification synchronously, Python's built-in `threading` module is used to run the task in the background:

```python
import threading
```
The synchronous operation:
```python
time.sleep(2)  # Simulate some work (e.g., sending email)
print(f"Notification sent for task {task_id}")

```
was replaced by a function called send_notification, which is executed in a separate thread:
```python
def send_notification(task_id):
    time.sleep(2)  # Simulate delay (e.g., sending an email)
    print(f"Notification sent for task {task_id}")
```
and then triggered like this:
```python
threading.Thread(target=send_notification, args=(task_id,), daemon=True).start()
```
Setting daemon=True ensures the thread runs as a background daemon, meaning it won't block the application from shutting down. This is ideal for short, auxiliary background tasks like sending notifications.

In the create_task route, the declaration of global next_task_id had to be moved before the variable was used to avoid a UnboundLocalError.
Changed from:
```python
new_task = {
    'id': next_task_id,
    ...
}
global next_task_id
```
to:
```python
global next_task_id
new_task = {
    'id': next_task_id,
    ...
}
```
## 2. RxJS Integration (Conceptual)

To support a reactive frontend using RxJS, the backend must be capable of sending real-time updates to clients. This allows the frontend to react to changes in task data (e.g., when a task is updated or completed) **without the need for polling**.

### Recommended Approach: WebSockets

To implement real-time, bidirectional communication between the backend and frontend, **WebSockets** are recommended over Server-Sent Events (SSE) due to their flexibility and support for full duplex (two-way) communication. This is particularly beneficial if the frontend should also send real-time events to the backend in the future.

#### Backend Design:

- Use the `flask-socketio` library to integrate WebSocket support into the Flask application.
- When a task is updated (e.g., via the `PUT /api/tasks/<id>` endpoint), emit a WebSocket event (e.g., `task_updated`) with the updated task payload.
- Clients connected via WebSockets can subscribe to specific event types and react accordingly using RxJS observables.

##### Example WebSocket event emission (Flask backend):

```python
from flask_socketio import SocketIO, emit

socketio = SocketIO(app, cors_allowed_origins="*")

@app.route('/api/tasks/<int:task_id>', methods=['PUT'])
def update_task(task_id):
    ...
    if task_updated:
        socketio.emit('task_updated', task)
        return jsonify(task), 200
```
Starting the app with WebSocket support:
```python
if __name__ == '__main__':
    socketio.run(app, debug=True)
```
#### Frontend Design (Angular + ngx-socket-io + RxJS):
On the frontend, we integrate WebSockets using ngx-socket-io and manage event streams using RxJS. This allows for a clean, Angular-friendly architecture where event updates are exposed via a service:
```ts
import { Injectable } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { Socket } from 'ngx-socket-io';

@Injectable({
  providedIn: 'root'
})
export class TaskService {
  private taskUpdates = new Subject<any>();

  constructor(private socket: Socket) {
    this.socket.on('task_updated', (data: any) => {
      this.taskUpdates.next(data);
    });
  }

  onTaskUpdate(): Observable<any> {
    return this.taskUpdates.asObservable();
  }

  disconnect(): void {
    this.socket.disconnect();
  }
}
```
Usage in component:
```ts
this.taskService.onTaskUpdate().subscribe(updatedTask => {
  // Update UI with new task data
});
```
### Why not Server-Sent Events (SSE)?

SSE is a good fit for simple use cases like notifications or live updates where data flows only from the server to the client. However, in this project, I prefer WebSockets because they provide bidirectional communication, which is useful not only for receiving updates about changes but also for creating and modifying tasks in real-time from the frontend. This offers more flexibility and better suits an interactive application.


### Key Concepts & Technologies
RxJS: Handles event streams using Subject and Observable for a reactive frontend architecture.

WebSockets: Enables real-time, two-way communication between client and server.

ngx-socket-io: Angular wrapper for socket.io-client, used to integrate WebSocket functionality easily into services.

flask-socketio: Flask extension to emit and listen for WebSocket events.

Event-driven architecture: The backend emits events like task_updated, and the frontend listens and reacts in real time.
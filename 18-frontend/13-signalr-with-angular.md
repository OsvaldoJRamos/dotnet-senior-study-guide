# SignalR with Angular

## Overview

SignalR provides real-time communication between ASP.NET Core backend and Angular frontend. The Angular client connects via WebSockets (with fallback to SSE/Long Polling) and exchanges messages through Hubs.

## Setup

### Install the client package

```bash
npm install @microsoft/signalr
```

### Create a SignalR service

```typescript
import { Injectable } from '@angular/core';
import * as signalR from '@microsoft/signalr';
import { BehaviorSubject, Subject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SignalRService {
  private hubConnection!: signalR.HubConnection;
  
  connectionState$ = new BehaviorSubject<string>('Disconnected');
  messageReceived$ = new Subject<{ user: string; message: string }>();

  connect(hubUrl: string): void {
    this.hubConnection = new signalR.HubConnectionBuilder()
      .withUrl(hubUrl, {
        accessTokenFactory: () => localStorage.getItem('token') ?? ''
      })
      .withAutomaticReconnect([0, 2000, 5000, 10000, 30000]) // retry intervals
      .configureLogging(signalR.LogLevel.Warning)
      .build();

    // Register event handlers BEFORE starting
    this.registerHandlers();

    this.hubConnection.start()
      .then(() => this.connectionState$.next('Connected'))
      .catch(err => console.error('SignalR connection error:', err));

    this.hubConnection.onreconnecting(() => 
      this.connectionState$.next('Reconnecting'));
    this.hubConnection.onreconnected(() => 
      this.connectionState$.next('Connected'));
    this.hubConnection.onclose(() => 
      this.connectionState$.next('Disconnected'));
  }

  private registerHandlers(): void {
    this.hubConnection.on('ReceiveMessage', (user: string, message: string) => {
      this.messageReceived$.next({ user, message });
    });
  }

  sendMessage(user: string, message: string): Promise<void> {
    return this.hubConnection.invoke('SendMessage', user, message);
  }

  joinGroup(groupName: string): Promise<void> {
    return this.hubConnection.invoke('JoinGroup', groupName);
  }

  disconnect(): void {
    this.hubConnection?.stop();
  }
}
```

## Using in a component

```typescript
@Component({
  selector: 'app-chat',
  template: `
    <div class="status">{{ connectionState$ | async }}</div>
    
    <div class="messages">
      <div *ngFor="let msg of messages">
        <strong>{{ msg.user }}:</strong> {{ msg.message }}
      </div>
    </div>

    <input [(ngModel)]="newMessage" (keyup.enter)="send()" />
    <button (click)="send()">Send</button>
  `
})
export class ChatComponent implements OnInit, OnDestroy {
  messages: { user: string; message: string }[] = [];
  newMessage = '';
  connectionState$ = this.signalR.connectionState$;

  private destroy$ = new Subject<void>();

  constructor(private signalR: SignalRService) {}

  ngOnInit(): void {
    this.signalR.connect('https://localhost:5001/chat');

    this.signalR.messageReceived$
      .pipe(takeUntil(this.destroy$))
      .subscribe(msg => this.messages.push(msg));
  }

  send(): void {
    if (!this.newMessage.trim()) return;
    this.signalR.sendMessage('User', this.newMessage);
    this.newMessage = '';
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.signalR.disconnect();
  }
}
```

## Backend Hub (ASP.NET Core)

```csharp
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }
}

// Program.cs
builder.Services.AddSignalR();
builder.Services.AddCors(options =>
    options.AddPolicy("angular", p => p
        .WithOrigins("http://localhost:4200")
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials())); // required for SignalR

app.UseCors("angular");
app.MapHub<ChatHub>("/chat");
```

> `AllowCredentials()` is **required** for SignalR with CORS — without it, the WebSocket handshake fails.

## Authentication with JWT

```typescript
// The accessTokenFactory in the connection builder handles this:
.withUrl(hubUrl, {
  accessTokenFactory: () => this.authService.getToken()
})
```

On the backend:

```csharp
app.MapHub<ChatHub>("/chat").RequireAuthorization();
```

SignalR sends the JWT as a query parameter for WebSockets (since WebSocket handshake doesn't support custom headers).

## Strongly-typed Hub (backend best practice)

```csharp
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
    Task UserLeft(string user);
}

public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.ReceiveMessage(user, message); // compile-time safety
    }
}
```

## Common patterns

### Real-time notifications

```typescript
// notification.service.ts
this.hubConnection.on('NewNotification', (notification: Notification) => {
  this.notifications$.next(notification);
  this.toastr.info(notification.message);
});
```

### Live dashboard updates

```typescript
// dashboard.component.ts
this.hubConnection.on('MetricsUpdated', (metrics: DashboardMetrics) => {
  this.metrics = metrics;
  this.chart.update(metrics); // update chart in real time
});
```

### Typing indicators

```typescript
// Debounce typing events to avoid flooding the server
this.typingSubject.pipe(
  debounceTime(300),
  distinctUntilChanged()
).subscribe(isTyping => {
  this.hubConnection.invoke('SetTypingStatus', this.userId, isTyping);
});
```

## Common pitfalls

1. **Register handlers before `.start()`** — messages sent during connection setup are lost otherwise
2. **Always handle reconnection** — `withAutomaticReconnect()` is not enabled by default
3. **Unsubscribe on destroy** — prevent memory leaks in Angular components
4. **CORS with credentials** — SignalR requires `AllowCredentials()` on the backend
5. **Zone.js** — SignalR callbacks run outside Angular's zone. Use `NgZone.run()` if change detection doesn't trigger:

```typescript
this.hubConnection.on('ReceiveMessage', (user, msg) => {
  this.ngZone.run(() => {
    this.messages.push({ user, message: msg });
  });
});
```

---

[← Previous: Angular Performance](12-angular-performance.md) | [Back to index](README.md)

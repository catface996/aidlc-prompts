# WebSocket 实时通信最佳实践

## 角色设定

你是一位精通 WebSocket 的前端实时通信专家，擅长实时消息、长连接管理和状态同步。

## 提示词模板

### WebSocket 实现

```
请帮我实现 WebSocket 功能：
- 使用场景：[聊天/通知/实时数据]
- 框架：[原生/Socket.io/SockJS]
- 重连策略：[是否需要]
- 心跳机制：[是否需要]

请提供完整的实现代码。
```

## 核心代码示例

### WebSocket 客户端封装

```typescript
// websocket.ts
type MessageHandler = (data: any) => void;
type ConnectionHandler = () => void;

interface WebSocketConfig {
  url: string;
  reconnect?: boolean;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  heartbeatInterval?: number;
}

class WebSocketClient {
  private ws: WebSocket | null = null;
  private config: Required<WebSocketConfig>;
  private messageHandlers: Map<string, Set<MessageHandler>> = new Map();
  private reconnectAttempts = 0;
  private heartbeatTimer: number | null = null;
  private reconnectTimer: number | null = null;
  private isManualClose = false;

  private onOpenHandlers: Set<ConnectionHandler> = new Set();
  private onCloseHandlers: Set<ConnectionHandler> = new Set();
  private onErrorHandlers: Set<(error: Event) => void> = new Set();

  constructor(config: WebSocketConfig) {
    this.config = {
      reconnect: true,
      reconnectInterval: 3000,
      maxReconnectAttempts: 10,
      heartbeatInterval: 30000,
      ...config,
    };
  }

  connect(): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      return;
    }

    this.isManualClose = false;
    this.ws = new WebSocket(this.config.url);

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
      this.startHeartbeat();
      this.onOpenHandlers.forEach((handler) => handler());
    };

    this.ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);
        const { type, data } = message;

        // 处理心跳响应
        if (type === 'pong') {
          return;
        }

        // 分发消息
        const handlers = this.messageHandlers.get(type);
        handlers?.forEach((handler) => handler(data));

        // 通用消息处理
        const allHandlers = this.messageHandlers.get('*');
        allHandlers?.forEach((handler) => handler(message));
      } catch (error) {
        console.error('Failed to parse message:', error);
      }
    };

    this.ws.onclose = () => {
      console.log('WebSocket disconnected');
      this.stopHeartbeat();
      this.onCloseHandlers.forEach((handler) => handler());

      if (!this.isManualClose && this.config.reconnect) {
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.onErrorHandlers.forEach((handler) => handler(error));
    };
  }

  disconnect(): void {
    this.isManualClose = true;
    this.stopHeartbeat();
    this.clearReconnectTimer();
    this.ws?.close();
    this.ws = null;
  }

  send(type: string, data: any): void {
    if (this.ws?.readyState !== WebSocket.OPEN) {
      console.warn('WebSocket is not connected');
      return;
    }

    this.ws.send(JSON.stringify({ type, data }));
  }

  on(type: string, handler: MessageHandler): () => void {
    if (!this.messageHandlers.has(type)) {
      this.messageHandlers.set(type, new Set());
    }
    this.messageHandlers.get(type)!.add(handler);

    return () => {
      this.messageHandlers.get(type)?.delete(handler);
    };
  }

  onOpen(handler: ConnectionHandler): () => void {
    this.onOpenHandlers.add(handler);
    return () => this.onOpenHandlers.delete(handler);
  }

  onClose(handler: ConnectionHandler): () => void {
    this.onCloseHandlers.add(handler);
    return () => this.onCloseHandlers.delete(handler);
  }

  onError(handler: (error: Event) => void): () => void {
    this.onErrorHandlers.add(handler);
    return () => this.onErrorHandlers.delete(handler);
  }

  get isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }

  private startHeartbeat(): void {
    this.heartbeatTimer = window.setInterval(() => {
      this.send('ping', { timestamp: Date.now() });
    }, this.config.heartbeatInterval);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  private scheduleReconnect(): void {
    if (this.reconnectAttempts >= this.config.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }

    this.reconnectTimer = window.setTimeout(() => {
      this.reconnectAttempts++;
      console.log(`Reconnecting... (${this.reconnectAttempts}/${this.config.maxReconnectAttempts})`);
      this.connect();
    }, this.config.reconnectInterval);
  }

  private clearReconnectTimer(): void {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }
  }
}

export const wsClient = new WebSocketClient({
  url: import.meta.env.VITE_WS_URL || 'ws://localhost:8080/ws',
});
```

### React Hook 封装

```typescript
// useWebSocket.ts
import { useEffect, useRef, useCallback, useState } from 'react';
import { wsClient } from './websocket';

interface UseWebSocketOptions {
  autoConnect?: boolean;
}

export function useWebSocket(options: UseWebSocketOptions = {}) {
  const { autoConnect = true } = options;
  const [isConnected, setIsConnected] = useState(false);
  const [lastMessage, setLastMessage] = useState<any>(null);

  useEffect(() => {
    if (autoConnect) {
      wsClient.connect();
    }

    const unsubOpen = wsClient.onOpen(() => setIsConnected(true));
    const unsubClose = wsClient.onClose(() => setIsConnected(false));

    return () => {
      unsubOpen();
      unsubClose();
    };
  }, [autoConnect]);

  const send = useCallback((type: string, data: any) => {
    wsClient.send(type, data);
  }, []);

  const subscribe = useCallback((type: string, handler: (data: any) => void) => {
    return wsClient.on(type, handler);
  }, []);

  return {
    isConnected,
    lastMessage,
    send,
    subscribe,
    connect: () => wsClient.connect(),
    disconnect: () => wsClient.disconnect(),
  };
}

// 订阅特定消息类型
export function useWebSocketMessage<T = any>(type: string) {
  const [message, setMessage] = useState<T | null>(null);

  useEffect(() => {
    const unsubscribe = wsClient.on(type, (data: T) => {
      setMessage(data);
    });

    return unsubscribe;
  }, [type]);

  return message;
}
```

### 聊天组件示例

```tsx
// Chat.tsx
import { useState, useEffect, useRef } from 'react';
import { useWebSocket, useWebSocketMessage } from './useWebSocket';

interface Message {
  id: string;
  userId: string;
  content: string;
  timestamp: number;
}

export function Chat({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputValue, setInputValue] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const { isConnected, send, subscribe } = useWebSocket();

  // 订阅新消息
  useEffect(() => {
    const unsubscribe = subscribe('chat:message', (data: Message) => {
      setMessages((prev) => [...prev, data]);
    });

    return unsubscribe;
  }, [subscribe]);

  // 加入房间
  useEffect(() => {
    if (isConnected) {
      send('chat:join', { roomId });
    }

    return () => {
      if (isConnected) {
        send('chat:leave', { roomId });
      }
    };
  }, [isConnected, roomId, send]);

  // 自动滚动
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSend = () => {
    if (!inputValue.trim()) return;

    send('chat:send', {
      roomId,
      content: inputValue,
    });

    setInputValue('');
  };

  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center justify-between p-4 border-b">
        <span>Room: {roomId}</span>
        <span className={`w-2 h-2 rounded-full ${isConnected ? 'bg-green-500' : 'bg-red-500'}`} />
      </div>

      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg) => (
          <div key={msg.id} className="flex flex-col">
            <span className="text-sm text-gray-500">{msg.userId}</span>
            <span className="bg-gray-100 rounded-lg p-2">{msg.content}</span>
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>

      <div className="p-4 border-t flex gap-2">
        <input
          type="text"
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleSend()}
          placeholder="Type a message..."
          className="flex-1 px-4 py-2 border rounded-lg"
          disabled={!isConnected}
        />
        <button
          onClick={handleSend}
          disabled={!isConnected}
          className="px-4 py-2 bg-blue-500 text-white rounded-lg disabled:opacity-50"
        >
          Send
        </button>
      </div>
    </div>
  );
}
```

### 实时通知

```tsx
// NotificationProvider.tsx
import { createContext, useContext, useEffect, useState } from 'react';
import { useWebSocket } from './useWebSocket';
import { toast } from 'react-hot-toast';

interface Notification {
  id: string;
  type: 'info' | 'success' | 'warning' | 'error';
  title: string;
  message: string;
  read: boolean;
  createdAt: number;
}

interface NotificationContextValue {
  notifications: Notification[];
  unreadCount: number;
  markAsRead: (id: string) => void;
  markAllAsRead: () => void;
}

const NotificationContext = createContext<NotificationContextValue | null>(null);

export function NotificationProvider({ children }: { children: React.ReactNode }) {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const { subscribe, isConnected } = useWebSocket();

  useEffect(() => {
    const unsubscribe = subscribe('notification', (data: Notification) => {
      setNotifications((prev) => [data, ...prev]);

      // 显示 toast
      toast[data.type](data.message, { duration: 5000 });
    });

    return unsubscribe;
  }, [subscribe]);

  const unreadCount = notifications.filter((n) => !n.read).length;

  const markAsRead = (id: string) => {
    setNotifications((prev) =>
      prev.map((n) => (n.id === id ? { ...n, read: true } : n))
    );
  };

  const markAllAsRead = () => {
    setNotifications((prev) => prev.map((n) => ({ ...n, read: true })));
  };

  return (
    <NotificationContext.Provider
      value={{ notifications, unreadCount, markAsRead, markAllAsRead }}
    >
      {children}
    </NotificationContext.Provider>
  );
}

export function useNotifications() {
  const context = useContext(NotificationContext);
  if (!context) {
    throw new Error('useNotifications must be used within NotificationProvider');
  }
  return context;
}
```

### Socket.IO 集成

```typescript
// socketio-client.ts
import { io, Socket } from 'socket.io-client';

interface SocketConfig {
  url: string;
  auth?: Record<string, string>;
}

class SocketIOClient {
  private socket: Socket | null = null;

  connect(config: SocketConfig): void {
    this.socket = io(config.url, {
      auth: config.auth,
      reconnection: true,
      reconnectionAttempts: 10,
      reconnectionDelay: 1000,
      timeout: 20000,
    });

    this.socket.on('connect', () => {
      console.log('Socket.IO connected:', this.socket?.id);
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Socket.IO disconnected:', reason);
    });

    this.socket.on('connect_error', (error) => {
      console.error('Socket.IO connection error:', error);
    });
  }

  disconnect(): void {
    this.socket?.disconnect();
    this.socket = null;
  }

  emit(event: string, data: any): void {
    this.socket?.emit(event, data);
  }

  on<T = any>(event: string, handler: (data: T) => void): () => void {
    this.socket?.on(event, handler);
    return () => this.socket?.off(event, handler);
  }

  once<T = any>(event: string, handler: (data: T) => void): void {
    this.socket?.once(event, handler);
  }

  get id(): string | undefined {
    return this.socket?.id;
  }

  get isConnected(): boolean {
    return this.socket?.connected ?? false;
  }
}

export const socketIO = new SocketIOClient();
```

## 最佳实践清单

- [ ] 实现自动重连机制
- [ ] 添加心跳保活
- [ ] 消息类型分发
- [ ] 连接状态管理
- [ ] 错误处理和日志
- [ ] 断线重连消息补偿
- [ ] 消息去重
- [ ] 性能优化（节流/防抖）

# Canvas 绘图最佳实践

## 角色设定

你是一位精通 Canvas 的前端可视化专家，擅长 2D 绘图、图表渲染和交互式图形。

## 提示词模板

### Canvas 绘图

```
请帮我实现 Canvas 绘图：
- 图形类型：[图表/游戏/编辑器/数据可视化]
- 交互需求：[点击/拖拽/缩放]
- 性能要求：[描述性能需求]
- 动画需求：[是否需要动画]

请提供完整的实现代码。
```

## 核心代码示例

### Canvas 基础封装

```typescript
// canvas/CanvasRenderer.ts
export class CanvasRenderer {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private dpr: number;
  private width: number;
  private height: number;

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d')!;
    this.dpr = window.devicePixelRatio || 1;
    this.resize();
  }

  resize(): void {
    const rect = this.canvas.getBoundingClientRect();
    this.width = rect.width;
    this.height = rect.height;

    // 高清屏适配
    this.canvas.width = this.width * this.dpr;
    this.canvas.height = this.height * this.dpr;
    this.ctx.scale(this.dpr, this.dpr);
  }

  clear(): void {
    this.ctx.clearRect(0, 0, this.width, this.height);
  }

  drawRect(x: number, y: number, width: number, height: number, options: DrawOptions = {}): void {
    this.ctx.save();
    this.applyOptions(options);

    if (options.fill) {
      this.ctx.fillRect(x, y, width, height);
    }
    if (options.stroke) {
      this.ctx.strokeRect(x, y, width, height);
    }

    this.ctx.restore();
  }

  drawCircle(x: number, y: number, radius: number, options: DrawOptions = {}): void {
    this.ctx.save();
    this.applyOptions(options);

    this.ctx.beginPath();
    this.ctx.arc(x, y, radius, 0, Math.PI * 2);

    if (options.fill) {
      this.ctx.fill();
    }
    if (options.stroke) {
      this.ctx.stroke();
    }

    this.ctx.restore();
  }

  drawLine(points: Point[], options: DrawOptions = {}): void {
    if (points.length < 2) return;

    this.ctx.save();
    this.applyOptions(options);

    this.ctx.beginPath();
    this.ctx.moveTo(points[0].x, points[0].y);

    for (let i = 1; i < points.length; i++) {
      this.ctx.lineTo(points[i].x, points[i].y);
    }

    this.ctx.stroke();
    this.ctx.restore();
  }

  drawText(text: string, x: number, y: number, options: TextOptions = {}): void {
    this.ctx.save();

    this.ctx.font = options.font || '14px sans-serif';
    this.ctx.fillStyle = options.color || '#000';
    this.ctx.textAlign = options.align || 'left';
    this.ctx.textBaseline = options.baseline || 'top';

    this.ctx.fillText(text, x, y);
    this.ctx.restore();
  }

  private applyOptions(options: DrawOptions): void {
    if (options.fillColor) {
      this.ctx.fillStyle = options.fillColor;
    }
    if (options.strokeColor) {
      this.ctx.strokeStyle = options.strokeColor;
    }
    if (options.lineWidth) {
      this.ctx.lineWidth = options.lineWidth;
    }
    if (options.lineDash) {
      this.ctx.setLineDash(options.lineDash);
    }
    if (options.shadowColor) {
      this.ctx.shadowColor = options.shadowColor;
      this.ctx.shadowBlur = options.shadowBlur || 0;
      this.ctx.shadowOffsetX = options.shadowOffsetX || 0;
      this.ctx.shadowOffsetY = options.shadowOffsetY || 0;
    }
  }
}

interface Point {
  x: number;
  y: number;
}

interface DrawOptions {
  fill?: boolean;
  stroke?: boolean;
  fillColor?: string;
  strokeColor?: string;
  lineWidth?: number;
  lineDash?: number[];
  shadowColor?: string;
  shadowBlur?: number;
  shadowOffsetX?: number;
  shadowOffsetY?: number;
}

interface TextOptions {
  font?: string;
  color?: string;
  align?: CanvasTextAlign;
  baseline?: CanvasTextBaseline;
}
```

### React Canvas Hook

```tsx
// hooks/useCanvas.ts
import { useRef, useEffect, useCallback } from 'react';

interface UseCanvasOptions {
  width?: number;
  height?: number;
  onDraw?: (ctx: CanvasRenderingContext2D) => void;
  animate?: boolean;
}

export function useCanvas(options: UseCanvasOptions = {}) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const animationRef = useRef<number>();

  const draw = useCallback(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    options.onDraw?.(ctx);

    if (options.animate) {
      animationRef.current = requestAnimationFrame(draw);
    }
  }, [options]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();

    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;

    const ctx = canvas.getContext('2d');
    ctx?.scale(dpr, dpr);

    draw();

    return () => {
      if (animationRef.current) {
        cancelAnimationFrame(animationRef.current);
      }
    };
  }, [draw]);

  return canvasRef;
}
```

### 折线图组件

```tsx
// components/LineChart.tsx
import { useRef, useEffect } from 'react';

interface DataPoint {
  x: number;
  y: number;
  label?: string;
}

interface LineChartProps {
  data: DataPoint[];
  width?: number;
  height?: number;
  padding?: number;
  lineColor?: string;
  pointColor?: string;
  gridColor?: string;
}

export function LineChart({
  data,
  width = 600,
  height = 400,
  padding = 40,
  lineColor = '#3b82f6',
  pointColor = '#3b82f6',
  gridColor = '#e5e7eb',
}: LineChartProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas || data.length === 0) return;

    const ctx = canvas.getContext('2d')!;
    const dpr = window.devicePixelRatio || 1;

    // 设置高清屏
    canvas.width = width * dpr;
    canvas.height = height * dpr;
    ctx.scale(dpr, dpr);

    // 计算数据范围
    const xValues = data.map((d) => d.x);
    const yValues = data.map((d) => d.y);
    const xMin = Math.min(...xValues);
    const xMax = Math.max(...xValues);
    const yMin = Math.min(...yValues);
    const yMax = Math.max(...yValues);

    const chartWidth = width - padding * 2;
    const chartHeight = height - padding * 2;

    // 坐标转换函数
    const toCanvasX = (x: number) =>
      padding + ((x - xMin) / (xMax - xMin)) * chartWidth;
    const toCanvasY = (y: number) =>
      height - padding - ((y - yMin) / (yMax - yMin)) * chartHeight;

    // 清除画布
    ctx.clearRect(0, 0, width, height);

    // 绘制网格
    ctx.strokeStyle = gridColor;
    ctx.lineWidth = 1;
    for (let i = 0; i <= 5; i++) {
      const y = padding + (chartHeight / 5) * i;
      ctx.beginPath();
      ctx.moveTo(padding, y);
      ctx.lineTo(width - padding, y);
      ctx.stroke();
    }

    // 绘制折线
    ctx.strokeStyle = lineColor;
    ctx.lineWidth = 2;
    ctx.beginPath();
    data.forEach((point, index) => {
      const x = toCanvasX(point.x);
      const y = toCanvasY(point.y);
      if (index === 0) {
        ctx.moveTo(x, y);
      } else {
        ctx.lineTo(x, y);
      }
    });
    ctx.stroke();

    // 绘制数据点
    ctx.fillStyle = pointColor;
    data.forEach((point) => {
      const x = toCanvasX(point.x);
      const y = toCanvasY(point.y);
      ctx.beginPath();
      ctx.arc(x, y, 4, 0, Math.PI * 2);
      ctx.fill();
    });

    // 绘制坐标轴
    ctx.strokeStyle = '#333';
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(padding, padding);
    ctx.lineTo(padding, height - padding);
    ctx.lineTo(width - padding, height - padding);
    ctx.stroke();

  }, [data, width, height, padding, lineColor, pointColor, gridColor]);

  return (
    <canvas
      ref={canvasRef}
      style={{ width, height }}
    />
  );
}
```

### 画板组件

```tsx
// components/DrawingBoard.tsx
import { useRef, useEffect, useState } from 'react';

interface DrawingBoardProps {
  width?: number;
  height?: number;
  strokeColor?: string;
  strokeWidth?: number;
}

export function DrawingBoard({
  width = 800,
  height = 600,
  strokeColor = '#000',
  strokeWidth = 2,
}: DrawingBoardProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [isDrawing, setIsDrawing] = useState(false);
  const [history, setHistory] = useState<ImageData[]>([]);
  const [historyIndex, setHistoryIndex] = useState(-1);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d')!;
    const dpr = window.devicePixelRatio || 1;

    canvas.width = width * dpr;
    canvas.height = height * dpr;
    ctx.scale(dpr, dpr);
    ctx.lineCap = 'round';
    ctx.lineJoin = 'round';

    // 保存初始状态
    saveState();
  }, [width, height]);

  const saveState = () => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d')!;
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    const newHistory = history.slice(0, historyIndex + 1);
    newHistory.push(imageData);
    setHistory(newHistory);
    setHistoryIndex(newHistory.length - 1);
  };

  const getCoords = (e: React.MouseEvent) => {
    const canvas = canvasRef.current!;
    const rect = canvas.getBoundingClientRect();
    return {
      x: e.clientX - rect.left,
      y: e.clientY - rect.top,
    };
  };

  const startDrawing = (e: React.MouseEvent) => {
    const ctx = canvasRef.current?.getContext('2d');
    if (!ctx) return;

    setIsDrawing(true);
    const { x, y } = getCoords(e);

    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.strokeStyle = strokeColor;
    ctx.lineWidth = strokeWidth;
  };

  const draw = (e: React.MouseEvent) => {
    if (!isDrawing) return;

    const ctx = canvasRef.current?.getContext('2d');
    if (!ctx) return;

    const { x, y } = getCoords(e);
    ctx.lineTo(x, y);
    ctx.stroke();
  };

  const stopDrawing = () => {
    if (isDrawing) {
      setIsDrawing(false);
      saveState();
    }
  };

  const undo = () => {
    if (historyIndex <= 0) return;

    const newIndex = historyIndex - 1;
    const ctx = canvasRef.current?.getContext('2d');
    if (!ctx) return;

    ctx.putImageData(history[newIndex], 0, 0);
    setHistoryIndex(newIndex);
  };

  const redo = () => {
    if (historyIndex >= history.length - 1) return;

    const newIndex = historyIndex + 1;
    const ctx = canvasRef.current?.getContext('2d');
    if (!ctx) return;

    ctx.putImageData(history[newIndex], 0, 0);
    setHistoryIndex(newIndex);
  };

  const clear = () => {
    const ctx = canvasRef.current?.getContext('2d');
    if (!ctx) return;

    ctx.clearRect(0, 0, width, height);
    saveState();
  };

  const download = () => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const link = document.createElement('a');
    link.download = 'drawing.png';
    link.href = canvas.toDataURL();
    link.click();
  };

  return (
    <div className="flex flex-col gap-4">
      <div className="flex gap-2">
        <button onClick={undo} disabled={historyIndex <= 0}>
          Undo
        </button>
        <button onClick={redo} disabled={historyIndex >= history.length - 1}>
          Redo
        </button>
        <button onClick={clear}>Clear</button>
        <button onClick={download}>Download</button>
      </div>
      <canvas
        ref={canvasRef}
        style={{ width, height, border: '1px solid #ccc', cursor: 'crosshair' }}
        onMouseDown={startDrawing}
        onMouseMove={draw}
        onMouseUp={stopDrawing}
        onMouseLeave={stopDrawing}
      />
    </div>
  );
}
```

### 粒子动画

```tsx
// components/ParticleCanvas.tsx
import { useRef, useEffect } from 'react';

interface Particle {
  x: number;
  y: number;
  vx: number;
  vy: number;
  radius: number;
  color: string;
}

export function ParticleCanvas({ width = 800, height = 600 }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const particlesRef = useRef<Particle[]>([]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d')!;
    const dpr = window.devicePixelRatio || 1;

    canvas.width = width * dpr;
    canvas.height = height * dpr;
    ctx.scale(dpr, dpr);

    // 创建粒子
    particlesRef.current = Array.from({ length: 100 }, () => ({
      x: Math.random() * width,
      y: Math.random() * height,
      vx: (Math.random() - 0.5) * 2,
      vy: (Math.random() - 0.5) * 2,
      radius: Math.random() * 3 + 1,
      color: `hsl(${Math.random() * 360}, 70%, 50%)`,
    }));

    let animationId: number;

    const animate = () => {
      ctx.clearRect(0, 0, width, height);

      particlesRef.current.forEach((particle) => {
        // 更新位置
        particle.x += particle.vx;
        particle.y += particle.vy;

        // 边界反弹
        if (particle.x <= 0 || particle.x >= width) particle.vx *= -1;
        if (particle.y <= 0 || particle.y >= height) particle.vy *= -1;

        // 绘制粒子
        ctx.beginPath();
        ctx.arc(particle.x, particle.y, particle.radius, 0, Math.PI * 2);
        ctx.fillStyle = particle.color;
        ctx.fill();
      });

      // 绘制连线
      ctx.strokeStyle = 'rgba(100, 100, 100, 0.1)';
      ctx.lineWidth = 1;
      particlesRef.current.forEach((p1, i) => {
        particlesRef.current.slice(i + 1).forEach((p2) => {
          const dx = p1.x - p2.x;
          const dy = p1.y - p2.y;
          const dist = Math.sqrt(dx * dx + dy * dy);

          if (dist < 100) {
            ctx.beginPath();
            ctx.moveTo(p1.x, p1.y);
            ctx.lineTo(p2.x, p2.y);
            ctx.stroke();
          }
        });
      });

      animationId = requestAnimationFrame(animate);
    };

    animate();

    return () => cancelAnimationFrame(animationId);
  }, [width, height]);

  return <canvas ref={canvasRef} style={{ width, height }} />;
}
```

## 性能优化

```typescript
// 离屏 Canvas
const offscreen = document.createElement('canvas');
const offCtx = offscreen.getContext('2d')!;
// 在离屏 canvas 预渲染复杂图形
ctx.drawImage(offscreen, 0, 0);

// 分层 Canvas
// 静态层 + 动态层分离

// 局部重绘
ctx.save();
ctx.beginPath();
ctx.rect(x, y, width, height);
ctx.clip();
// 只重绘裁剪区域
ctx.restore();

// 使用 requestAnimationFrame
function animate() {
  // 更新和渲染
  requestAnimationFrame(animate);
}
```

## 最佳实践清单

- [ ] 高清屏适配 (devicePixelRatio)
- [ ] 使用离屏 Canvas 缓存
- [ ] 分层渲染优化
- [ ] 局部重绘减少开销
- [ ] 使用 requestAnimationFrame
- [ ] 合理管理状态保存/恢复
- [ ] 事件节流处理
- [ ] 内存管理和清理

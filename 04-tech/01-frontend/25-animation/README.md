# Animation 动画最佳实践

## 角色设定

你是一位精通前端动画的专家，擅长 CSS 动画、Framer Motion、GSAP 和性能优化。

## 提示词模板

### 动画实现

```
请帮我实现动画效果：
- 动画类型：[过渡/关键帧/交互/滚动]
- 技术选择：[CSS/Framer Motion/GSAP]
- 性能要求：[60fps/低功耗]
- 具体效果：[描述动画效果]

请提供完整的实现代码。
```

## 核心代码示例

### CSS 动画

```css
/* animations.css */

/* 淡入淡出 */
.fade-enter {
  opacity: 0;
}
.fade-enter-active {
  opacity: 1;
  transition: opacity 300ms ease-in;
}
.fade-exit {
  opacity: 1;
}
.fade-exit-active {
  opacity: 0;
  transition: opacity 300ms ease-out;
}

/* 滑入滑出 */
.slide-enter {
  transform: translateX(-100%);
}
.slide-enter-active {
  transform: translateX(0);
  transition: transform 300ms ease-out;
}
.slide-exit {
  transform: translateX(0);
}
.slide-exit-active {
  transform: translateX(100%);
  transition: transform 300ms ease-in;
}

/* 缩放弹跳 */
@keyframes bounce {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.bounce {
  animation: bounce 0.5s ease-in-out;
}

/* 脉冲效果 */
@keyframes pulse {
  0% { transform: scale(1); opacity: 1; }
  50% { transform: scale(1.05); opacity: 0.8; }
  100% { transform: scale(1); opacity: 1; }
}

.pulse {
  animation: pulse 2s infinite ease-in-out;
}

/* 旋转加载 */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.spinner {
  animation: spin 1s linear infinite;
}

/* 抖动效果 */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  10%, 30%, 50%, 70%, 90% { transform: translateX(-5px); }
  20%, 40%, 60%, 80% { transform: translateX(5px); }
}

.shake {
  animation: shake 0.5s ease-in-out;
}

/* 骨架屏闪烁 */
@keyframes skeleton {
  0% { background-position: -200px 0; }
  100% { background-position: calc(200px + 100%) 0; }
}

.skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200px 100%;
  animation: skeleton 1.5s infinite;
}
```

### Framer Motion 基础

```tsx
// components/AnimatedComponents.tsx
import { motion, AnimatePresence } from 'framer-motion';

// 淡入组件
export function FadeIn({ children, delay = 0 }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.5, delay }}
    >
      {children}
    </motion.div>
  );
}

// 列表动画
export function AnimatedList({ items }) {
  return (
    <motion.ul>
      {items.map((item, index) => (
        <motion.li
          key={item.id}
          initial={{ opacity: 0, x: -20 }}
          animate={{ opacity: 1, x: 0 }}
          transition={{ delay: index * 0.1 }}
          whileHover={{ scale: 1.02 }}
          whileTap={{ scale: 0.98 }}
        >
          {item.content}
        </motion.li>
      ))}
    </motion.ul>
  );
}

// 页面过渡
export function PageTransition({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0, x: -20 }}
      animate={{ opacity: 1, x: 0 }}
      exit={{ opacity: 0, x: 20 }}
      transition={{ duration: 0.3 }}
    >
      {children}
    </motion.div>
  );
}

// 模态框动画
export function AnimatedModal({ isOpen, onClose, children }) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            className="fixed inset-0 bg-black/50 z-40"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          <motion.div
            className="fixed inset-0 flex items-center justify-center z-50"
            initial={{ opacity: 0, scale: 0.9 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.9 }}
            transition={{ type: 'spring', damping: 25, stiffness: 300 }}
          >
            <div className="bg-white rounded-lg p-6 max-w-md w-full">
              {children}
            </div>
          </motion.div>
        </>
      )}
    </AnimatePresence>
  );
}
```

### Framer Motion 高级

```tsx
// components/AdvancedAnimations.tsx
import { motion, useScroll, useTransform, useSpring } from 'framer-motion';
import { useRef } from 'react';

// 滚动视差
export function ParallaxSection({ children }) {
  const ref = useRef(null);
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start end', 'end start'],
  });

  const y = useTransform(scrollYProgress, [0, 1], [100, -100]);
  const opacity = useTransform(scrollYProgress, [0, 0.5, 1], [0, 1, 0]);

  return (
    <motion.section ref={ref} style={{ y, opacity }}>
      {children}
    </motion.section>
  );
}

// 拖拽排序
export function DraggableList({ items, onReorder }) {
  return (
    <motion.ul layout className="space-y-2">
      {items.map((item) => (
        <motion.li
          key={item.id}
          layout
          drag="y"
          dragConstraints={{ top: 0, bottom: 0 }}
          dragElastic={0.1}
          whileDrag={{ scale: 1.02, boxShadow: '0 10px 30px rgba(0,0,0,0.1)' }}
          className="bg-white p-4 rounded-lg cursor-grab active:cursor-grabbing"
        >
          {item.content}
        </motion.li>
      ))}
    </motion.ul>
  );
}

// 手势动画
export function GestureCard({ children }) {
  return (
    <motion.div
      className="bg-white p-6 rounded-xl shadow-lg"
      whileHover={{ scale: 1.02, rotateY: 5 }}
      whileTap={{ scale: 0.98 }}
      drag
      dragConstraints={{ left: -100, right: 100, top: -50, bottom: 50 }}
      dragElastic={0.2}
      style={{ perspective: 1000 }}
    >
      {children}
    </motion.div>
  );
}

// 路径动画
export function PathAnimation() {
  return (
    <motion.svg width="200" height="200" viewBox="0 0 200 200">
      <motion.path
        d="M 10 80 Q 95 10 180 80"
        fill="transparent"
        stroke="#0066ff"
        strokeWidth="3"
        initial={{ pathLength: 0 }}
        animate={{ pathLength: 1 }}
        transition={{ duration: 2, ease: 'easeInOut' }}
      />
    </motion.svg>
  );
}

// 数字计数动画
export function AnimatedCounter({ value }) {
  const spring = useSpring(0, { stiffness: 100, damping: 30 });

  useEffect(() => {
    spring.set(value);
  }, [value, spring]);

  return (
    <motion.span>
      {spring}
    </motion.span>
  );
}
```

### GSAP 动画

```tsx
// components/GSAPAnimations.tsx
import { useEffect, useRef } from 'react';
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';

gsap.registerPlugin(ScrollTrigger);

// 文字逐字动画
export function TextReveal({ text }) {
  const containerRef = useRef(null);

  useEffect(() => {
    const chars = containerRef.current.querySelectorAll('.char');

    gsap.fromTo(
      chars,
      { opacity: 0, y: 50 },
      {
        opacity: 1,
        y: 0,
        stagger: 0.05,
        duration: 0.5,
        ease: 'power2.out',
      }
    );
  }, []);

  return (
    <div ref={containerRef}>
      {text.split('').map((char, i) => (
        <span key={i} className="char inline-block">
          {char === ' ' ? '\u00A0' : char}
        </span>
      ))}
    </div>
  );
}

// 滚动触发动画
export function ScrollReveal({ children }) {
  const ref = useRef(null);

  useEffect(() => {
    gsap.fromTo(
      ref.current,
      { opacity: 0, y: 100 },
      {
        opacity: 1,
        y: 0,
        duration: 1,
        scrollTrigger: {
          trigger: ref.current,
          start: 'top 80%',
          end: 'top 50%',
          toggleActions: 'play none none reverse',
        },
      }
    );
  }, []);

  return <div ref={ref}>{children}</div>;
}

// 时间轴动画
export function Timeline() {
  const containerRef = useRef(null);

  useEffect(() => {
    const tl = gsap.timeline({ defaults: { duration: 0.5 } });

    tl.from('.box-1', { x: -100, opacity: 0 })
      .from('.box-2', { y: 100, opacity: 0 }, '-=0.3')
      .from('.box-3', { x: 100, opacity: 0 }, '-=0.3')
      .from('.title', { scale: 0, opacity: 0 });

    return () => tl.kill();
  }, []);

  return (
    <div ref={containerRef}>
      <div className="box-1">Box 1</div>
      <div className="box-2">Box 2</div>
      <div className="box-3">Box 3</div>
      <h1 className="title">Title</h1>
    </div>
  );
}

// 固定滚动动画
export function PinnedSection() {
  const ref = useRef(null);

  useEffect(() => {
    gsap.to('.progress', {
      width: '100%',
      scrollTrigger: {
        trigger: ref.current,
        start: 'top top',
        end: 'bottom bottom',
        pin: true,
        scrub: 1,
      },
    });
  }, []);

  return (
    <div ref={ref} className="h-[200vh]">
      <div className="sticky top-0 h-screen flex items-center justify-center">
        <div className="w-full h-2 bg-gray-200">
          <div className="progress h-full bg-blue-500 w-0" />
        </div>
      </div>
    </div>
  );
}
```

### React Spring

```tsx
// components/SpringAnimations.tsx
import { useSpring, useSprings, animated, useTrail } from '@react-spring/web';

// 基础弹簧动画
export function SpringBox({ isActive }) {
  const spring = useSpring({
    transform: isActive ? 'scale(1.1)' : 'scale(1)',
    backgroundColor: isActive ? '#3b82f6' : '#e5e7eb',
  });

  return <animated.div style={spring} className="w-20 h-20 rounded-lg" />;
}

// 交错动画
export function TrailList({ items, open }) {
  const trail = useTrail(items.length, {
    opacity: open ? 1 : 0,
    transform: open ? 'translateY(0)' : 'translateY(20px)',
    config: { mass: 1, tension: 280, friction: 60 },
  });

  return (
    <ul>
      {trail.map((style, index) => (
        <animated.li key={items[index].id} style={style}>
          {items[index].content}
        </animated.li>
      ))}
    </ul>
  );
}

// 卡片翻转
export function FlipCard({ front, back }) {
  const [flipped, setFlipped] = useState(false);

  const { transform, opacity } = useSpring({
    transform: `perspective(600px) rotateY(${flipped ? 180 : 0}deg)`,
    opacity: flipped ? 0 : 1,
    config: { mass: 5, tension: 500, friction: 80 },
  });

  return (
    <div className="relative w-64 h-40" onClick={() => setFlipped(!flipped)}>
      <animated.div
        className="absolute w-full h-full bg-blue-500 rounded-lg"
        style={{ opacity, transform }}
      >
        {front}
      </animated.div>
      <animated.div
        className="absolute w-full h-full bg-green-500 rounded-lg"
        style={{
          opacity: opacity.to((o) => 1 - o),
          transform: transform.to((t) => `${t} rotateY(180deg)`),
        }}
      >
        {back}
      </animated.div>
    </div>
  );
}
```

### 性能优化

```tsx
// 使用 will-change
const optimizedStyle = {
  willChange: 'transform, opacity',
  transform: 'translateZ(0)', // 开启 GPU 加速
};

// 使用 CSS containment
const containedStyle = {
  contain: 'layout paint',
};

// 减少重排的动画
// ✅ 好：使用 transform 和 opacity
transform: 'translateX(100px)';
opacity: 0.5;

// ❌ 差：触发重排的属性
left: '100px';
width: '200px';
```

## 最佳实践清单

- [ ] 优先使用 transform 和 opacity
- [ ] 开启 GPU 加速
- [ ] 使用 requestAnimationFrame
- [ ] 避免同时动画过多元素
- [ ] 使用 will-change 提示浏览器
- [ ] 减少重排重绘
- [ ] 设置合理的动画时长 (200-500ms)
- [ ] 支持 prefers-reduced-motion

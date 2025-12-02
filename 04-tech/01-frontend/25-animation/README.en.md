# Animation Best Practices

## 1. Role Definition

You are a performance optimization expert specializing in frontend animations, focused on creating smooth user experiences using CSS animations, Framer Motion, GSAP, and React Spring. You deeply understand browser rendering mechanisms, animation performance optimization, and user perception principles, and can find the optimal balance between visual effects and performance.

## 2. Core Principles (NON-NEGOTIABLE)

| Principle | Description | Consequences of Violation |
|------|------|----------|
| Performance First | Only use transform and opacity properties for animations, avoid triggering layout reflow and repaint | Animation stuttering, frame rate drop, poor user experience, non-smooth page scrolling |
| 60fps Standard | Ensure all animations maintain 60 frames per second, single frame time not exceeding 16.67ms | Obvious dropped frames and jitter in animations, affecting brand image |
| GPU Acceleration | Enable hardware acceleration for complex animations using transform3d or translateZ | High CPU load, severe mobile device heating, rapid battery consumption |
| Accessibility Support | MUST respect user's prefers-reduced-motion settings | Causes user dizziness and discomfort, violates accessibility standards, potential legal risks |
| Reasonable Duration | Micro-interactions 200-300ms, page transitions 300-500ms, loading animations 1-2s | Too short feels abrupt, too long causes user anxiety, reduces perceived performance |
| Progressive Enhancement | Animation failure should not affect core functionality, ensure content accessibility | Animation library load failure makes page unusable, losing core business |
| Avoid Simultaneous Animations | No more than 5 animated elements per screen | Browser main thread blocking, page becomes unresponsive, user cannot operate |
| Memory Management | Clean up animation instances and event listeners promptly | Memory leaks, page crashes after prolonged use |

## 3. Prompt Templates

### Template 1: Basic Animation Implementation

```
Please implement the following animation effect:

【Animation Type】
- Type: [fade in/out/slide/scale/rotate/bounce/shake/pulse]
- Trigger Method: [page load/user click/scroll into viewport/mouse hover]
- Duration: [specific duration in 200-500ms range]

【Technical Choice】
- Primary Solution: {CSS Animation/Framer Motion/GSAP/React Spring}
- Backup Solution: {alternative tech stack}
- Selection Reason: {based on project complexity, team skills, bundle size considerations}

【Performance Constraints】
- Target Frame Rate: 60fps
- GPU Acceleration Needed: {yes/no}
- Max Concurrent Animations: {number}

【Accessibility Requirements】
- Support prefers-reduced-motion: yes
- Fallback Plan: {alternative effect when animations disabled}

Please provide:
1. Animation implementation approach (textual description)
2. Performance optimization strategy
3. Accessibility fallback plan
4. Browser compatibility notes
```

### Template 2: Complex Interactive Animation

```
Please implement complex interactive animation:

【Interaction Scenario】
- Scenario Description: {drag sort/gesture swipe/parallax scroll/path animation}
- User Operation: {how users trigger and control animation}
- Expected Feedback: {what effect users should perceive}

【Animation Orchestration】
- Animation Stages: {initial state → transition state → end state}
- Timeline Arrangement: {timing relationship of each stage}
- Easing Function: {ease-in/ease-out/ease-in-out/spring/custom}

【Technical Selection】
- Recommended Technology: {based on interaction complexity}
- Dependency Library Version: {specific version number}
- Bundle Impact: {estimated KB increase}

【Performance Boundaries】
- Max Draggable Elements: {number limit}
- Scroll Listener Throttle: {throttle interval ms}
- Animation Cancel Conditions: {when to interrupt animation}

Please provide:
1. Interaction state machine design
2. Animation timeline planning
3. Gesture recognition strategy
4. Performance monitoring plan
```

### Template 3: Page Transition Animation

```
Please design page transition animation:

【Transition Scenario】
- From Page: {source page type}
- To Page: {target page type}
- Navigation Method: {forward/backward/replace}

【Transition Effect】
- Enter Animation: {fade in/slide in/scale in/rotate in}
- Exit Animation: {fade out/slide out/scale out/rotate out}
- Shared Elements: {whether elements need transition between pages}

【User Experience】
- Transition Duration: {suggest 300-500ms}
- Loading State: {how to display async loading}
- Interruption Handling: {how to handle rapid user clicks}

【Router Integration】
- Router Solution: {React Router/Next.js/custom}
- Integration Method: {describe how to work with routing system}

Please provide:
1. Transition animation design plan
2. Router integration strategy
3. Loading state handling
4. Exception scenario fallback
```

### Template 4: Scroll-Driven Animation

```
Please implement scroll-driven animation:

【Scroll Effect】
- Effect Type: {parallax scroll/element fade in/progress indicator/fixed positioning switch}
- Trigger Area: {scroll percentage from start to end position}
- Animation Relationship: {mapping relationship between scroll distance and animation progress}

【Scroll Listening】
- Listening Method: {Intersection Observer/Scroll Event/CSS Scroll-Driven}
- Performance Optimization: {use debounce throttle/requestAnimationFrame}
- Listening Scope: {global scroll/local container}

【Responsive Adaptation】
- Mobile Performance: {animation effect on touch scroll}
- Breakpoint Adjustments: {differences across screen sizes}

【Boundary Handling】
- Initial State: {animation state when page loads}
- Fast Scrolling: {performance protection during high-speed scroll}

Please provide:
1. Scroll listening solution
2. Animation progress calculation logic
3. Performance optimization measures
4. Mobile adaptation strategy
```

### Template 5: Loading & Skeleton Screen Animation

```
Please design loading animation solution:

【Loading Scenario】
- Scenario Type: {first screen load/data request/component lazy load/image load}
- Wait Duration: {expected loading time range}
- Failure Handling: {display plan when loading fails}

【Animation Type】
- Loader Style: {spinner/progress bar/pulse dots/skeleton screen}
- Animation Loop: {infinite loop/finite count}
- Transition Effect: {switching animation after load complete}

【User Perception】
- Psychological Expectation: {how to reduce user wait anxiety}
- Progress Display: {whether to show percentage or stages}
- Cancel Mechanism: {whether user can interrupt loading}

【Performance Considerations】
- Animation Lightweight: {avoid complex animations increasing load time}
- Lazy Load Strategy: {when to start animation, when to stop}

Please provide:
1. Loading animation design plan
2. Skeleton screen structure planning
3. Loading state management
4. Timeout and error handling
```

## 4. Decision Guide

### Animation Technology Selection Decision Tree

```
Animation Requirement
├─ Is it a simple transition effect (fade/move/scale)?
│  ├─ Yes → Use CSS Animation
│  │      ├─ Advantages: Best performance, no JS needed, zero bundle size, hardware acceleration
│  │      ├─ Use Cases:
│  │      │   ├─ Hover state switching
│  │      │   ├─ Page element fade in
│  │      │   ├─ Loading indicator rotation
│  │      │   └─ Button click feedback
│  │      └─ Limitations: Cannot respond to complex interaction logic
│  │
│  └─ No → Continue judging
│
├─ Need complex gesture interaction (drag/swipe/pinch)?
│  ├─ Yes → Use Framer Motion
│  │      ├─ Advantages: Declarative API, comprehensive gesture support, layout animations
│  │      ├─ Use Cases:
│  │      │   ├─ Drag sort list
│  │      │   ├─ Card swipe to delete
│  │      │   ├─ Modal drag to close
│  │      │   └─ Shared element transitions
│  │      ├─ Bundle Size: ~40KB gzipped
│  │      └─ Integration: Deep integration with React
│  │
│  └─ No → Continue judging
│
├─ Need precise timeline control and complex animation sequences?
│  ├─ Yes → Use GSAP
│  │      ├─ Advantages: Most powerful features, precise timeline, rich plugins
│  │      ├─ Use Cases:
│  │      │   ├─ Complex page entrance animations
│  │      │   ├─ SVG path animations
│  │      │   ├─ Scroll-triggered choreographed animations
│  │      │   └─ Text character-by-character reveal
│  │      ├─ Bundle Size: Core ~50KB, plugins on-demand
│  │      ├─ Performance: Highly optimized, supports mass element animation
│  │      └─ Learning Curve: Relatively steep, but comprehensive features
│  │
│  └─ No → Continue judging
│
└─ Pursuing extreme physics-based spring effects?
   ├─ Yes → Use React Spring
   │      ├─ Advantages: Physics-based spring animations, powerful interpolation
   │      ├─ Use Cases:
   │      │   ├─ Elastic card interactions
   │      │   ├─ Natural value transitions
   │      │   ├─ Staggered list item animations
   │      │   └─ State-driven continuous animations
   │      ├─ Bundle Size: ~25KB gzipped
   │      └─ Features: Declarative hooks API, interruptible animations
   │
   └─ No → Use combination
          ├─ CSS for simple transitions
          ├─ Framer Motion for interactions
          └─ Import other libraries as needed
```

### Performance Optimization Decision Tree

```
Animation Performance Issues
├─ Obvious stuttering (fps below 30)?
│  ├─ Yes → Check animation properties
│  │      ├─ Using left/top/width/height?
│  │      │  └─ Yes → Use transform and opacity instead
│  │      ├─ Animating more than 10 elements simultaneously?
│  │      │  └─ Yes → Reduce animated elements or use staggered delays
│  │      └─ Triggering large area repaints?
│  │         └─ Yes → Use will-change or contain property
│  │
│  └─ No → Continue judging
│
├─ Severe mobile device heating?
│  ├─ Yes → Check GPU usage
│  │      ├─ Hardware acceleration enabled?
│  │      │  └─ No → Add transform: translateZ(0)
│  │      ├─ Animation always running?
│  │      │  └─ Yes → Pause animation when not visible
│  │      └─ Using complex filter or backdrop-filter?
│  │         └─ Yes → Simplify effects or use static alternative
│  │
│  └─ No → Continue judging
│
├─ Animation delay on first load?
│  ├─ Yes → Optimize loading strategy
│  │      ├─ Animation library loaded on-demand?
│  │      │  └─ No → Use dynamic import and code splitting
│  │      ├─ Waiting for all resources to load?
│  │      │  └─ Yes → Use skeleton screen to render placeholder first
│  │      └─ Animation library bundle too large?
│  │         └─ Yes → Switch to lighter solution or import only necessary modules
│  │
│  └─ No → Continue judging
│
└─ Animation stuttering during scroll?
   ├─ Yes → Optimize scroll listening
   │      ├─ Using scroll event listener?
   │      │  └─ Yes → Use Intersection Observer instead
   │      ├─ Added throttle/debounce?
   │      │  └─ No → Add requestAnimationFrame throttle
   │      └─ Triggering layout calculation during scroll?
   │         └─ Yes → Cache calculation results, avoid forced synchronous layout
   │
   └─ No → Performance is adequate, consider visual optimization
```

## 5. Positive and Negative Examples

### Animation Property Selection

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|-----------|-----------|
| Horizontal Movement | Use left or margin-left property, triggers layout reflow, poor performance | Use transform: translateX(), only triggers composite, better performance |
| Vertical Movement | Use top or margin-top property, blocks main thread | Use transform: translateY(), GPU accelerated |
| Element Scaling | Modify width and height, causes child element recalculation | Use transform: scale(), doesn't affect layout |
| Opacity Change | Correctly use opacity property (both same) | Correctly use opacity property (both same) |
| Element Rotation | Use CSS transform rotate simulation, but calculate angles in JS | Directly use transform: rotate() or animation library |
| Color Transition | Frequently modifying backgroundColor with many elements causes poor performance | Few elements can transition directly, many elements consider using opacity mask |

### Performance Optimization Strategy

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|-----------|-----------|
| GPU Acceleration | No hints added, rely on browser auto-judgment | Add transform: translateZ(0) or will-change: transform |
| will-change Usage | Permanently set will-change on all elements, wasting memory | Only add before animation starts, remove after completion |
| Animation Count | Start 50 element animations simultaneously on page load | Use staggered delays, animate only 5-10 elements at a time |
| Scroll Listening | Directly listen to scroll events, perform calculations every scroll | Use Intersection Observer or add throttling |
| Animation Cleanup | Don't clean up animation instances and timers when component unmounts | Clean up in useEffect cleanup or componentWillUnmount |
| Memory Management | Keep references to all animated elements even when invisible | Pause or destroy animations when elements leave viewport |

### User Experience Design

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|-----------|-----------|
| Animation Duration | Micro-interactions over 1 second, makes users feel sluggish | Micro-interactions 200-300ms, quick response |
| Page Transition | Page switch animation over 800ms, user anxiety | Page transitions 300-500ms, smooth and natural |
| Loading Animation | Infinite spin with no feedback, user doesn't know progress | Show progress percentage or stage prompts |
| Accessibility | Ignore prefers-reduced-motion, force animations | Detect settings, provide static or simplified version |
| Easing Function | All animations use linear, feels mechanical | Choose ease-in-out/spring based on scenario |
| Animation Interruption | Animations stack or become chaotic with rapid user clicks | Interrupt current animation, immediately switch to new state |

### Technical Selection Decision

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|-----------|-----------|
| Simple Fade In | Import 60KB animation library for basic fade effect | Use CSS transition, zero cost |
| Complex Choreography | Use CSS animation for 10-step timeline, hard to maintain | Use GSAP Timeline, clear code |
| Drag Interaction | Implement all gesture recognition logic yourself, many bugs | Use Framer Motion's drag functionality |
| Scroll Parallax | Listen to scroll events, manually calculate all positions | Use GSAP ScrollTrigger plugin |
| Physics Animation | Use easing functions to simulate spring effect, unnatural | Use React Spring's physics engine |
| Router Transitions | Repeat animation logic in every page component | Use Framer Motion's AnimatePresence |

### Code Organization

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|-----------|-----------|
| Animation Reuse | Write repeated animation configuration in each component | Extract animation config to constant objects or custom hooks |
| Configuration Management | Hard-code animation parameters inside components | Use config file to centrally manage duration, easing, etc. |
| Conditional Animation | Use complex if-else to judge different state animations | Use state machine or variants pattern |
| Animation Composition | Mix multiple animation logic in one component | Split into independent animation components or hooks |
| Testing | Don't write tests for animations, rely on manual verification | Test animation trigger conditions and state transition logic |
| Documentation | Don't document reasons for animation parameter choices | Comment explaining reasons for duration, easing choices |

## 6. Validation Checklist

### Performance Validation

- [ ] Use Chrome DevTools Performance panel to record animation, confirm frame rate stable at 60fps
- [ ] Check for green Composite Layers, confirm GPU acceleration is active
- [ ] View Layers panel, confirm animated elements promoted to independent layers
- [ ] Use Rendering panel's Paint Flashing, confirm no unnecessary repaints
- [ ] Use Layout Shift check, confirm no unexpected layout shifts
- [ ] Test on mobile devices, use CPU and GPU throttling to simulate low-end devices
- [ ] Check Memory panel, confirm memory properly released after animation ends
- [ ] Run animations long-term, observe for memory leaks

### User Experience Validation

- [ ] All micro-interactions respond within 200-300ms
- [ ] Page transition animations last within 300-500ms range
- [ ] Loading animations show progress or hints after 2 seconds
- [ ] Animations don't stack or become chaotic with rapid consecutive clicks
- [ ] Animations can smoothly transition when interrupted
- [ ] First Input Delay (FID) less than 100ms
- [ ] Animations don't obscure important action buttons
- [ ] Loading animations don't cause user wait anxiety exceeding 3 seconds

### Accessibility Validation

- [ ] Detect prefers-reduced-motion media query
- [ ] Enable reduced motion in system settings, verify fallback plan
- [ ] Content still accessible and readable when animation fails
- [ ] Focus management not affected by animations, keyboard navigation normal
- [ ] Screen readers can correctly announce state changes
- [ ] Animations don't trigger photosensitive epilepsy (avoid rapid flashing)
- [ ] Provide option to pause or stop auto-playing animations

### Compatibility Validation

- [ ] Test on latest versions of Chrome, Firefox, Safari, Edge
- [ ] Test on iOS Safari and Android Chrome mobile browsers
- [ ] Test fallback plan for older browsers (if not supporting certain CSS features)
- [ ] Verify CSS prefixes correctly added (using Autoprefixer)
- [ ] Check animation library browser compatibility claims
- [ ] Test behavior in React strict mode
- [ ] Verify animation behavior in server-side rendering scenarios

### Code Quality Validation

- [ ] Animation components extracted as reusable modules
- [ ] Animation configuration parameters centrally managed
- [ ] All animation instances properly cleaned up when component unmounts
- [ ] Event listeners correctly added and removed
- [ ] Animation-related side effects properly handled in useEffect
- [ ] Added necessary code comments explaining animation intent
- [ ] Passes ESLint check, no animation-related warnings
- [ ] TypeScript type definitions complete (if applicable)

### Bundle Optimization Validation

- [ ] Animation libraries use on-demand import, avoid bundling entire library
- [ ] Use webpack-bundle-analyzer to analyze animation library size
- [ ] Non-critical animation libraries use dynamic import (lazy loading)
- [ ] CSS animations extracted to independent file and compressed
- [ ] Tree-shaking working correctly, unused animation code removed
- [ ] Production build enables code compression and minification
- [ ] Critical animation-related code inlined in main bundle
- [ ] Non-critical animation code split into independent chunks

## 7. Guardrail Constraints

### Performance Constraint Rules

| Constraint Type | Hard Limit | Explanation | Monitoring Method |
|---------|---------|------|---------|
| Frame Rate Lower Limit | Not below 30fps | Below this value causes obvious stuttering, must optimize | Chrome DevTools Performance panel |
| Target Frame Rate | Stable 60fps | Standard smooth experience for modern devices | Record 6 seconds of animation, view frame rate chart |
| Single Frame Budget | Not exceeding 16.67ms | 60fps requires each frame complete within 16.67ms | Performance panel view per-frame time |
| Concurrent Animation Count | Not exceeding 5 | Exceeding will block main thread | Manual review or custom performance monitoring |
| will-change Count | Not exceeding 3 | Too many consumes large memory | Review CSS code |
| GPU Layer Count | Not exceeding 10 layers | Too many composite layers consume GPU memory | Layers panel view layer tree |
| Animation Library Size | Single library not exceeding 50KB | Avoid excessive first screen load time | Bundle analyzer analysis |
| First Screen Animation Delay | Not exceeding 100ms | User perception instant feedback threshold | Lighthouse test First Input Delay |

### User Experience Constraints

| Constraint Type | Hard Limit | Explanation | Verification Method |
|---------|---------|------|---------|
| Micro-interaction Duration | 200-300ms | Feedback time for buttons, links and small elements | Use stopwatch or screen recording measurement |
| Transition Animation Duration | 300-500ms | Page switches, modals and medium-scale animations | Developer tools slow mode verification |
| Loading Animation Duration | 1-2 seconds per cycle | Avoid user anxiety, but not too fast | Simulate slow network testing |
| Max Wait Time | Show progress after 3 seconds | Must provide progress feedback beyond 3 seconds | Set network delay testing |
| Animation Interrupt Response | Immediate response | User operations must immediately interrupt current animation | Rapid consecutive click testing |
| Anxiety Threshold | Animations not exceeding 800ms | User feels obvious waiting beyond this | User testing feedback |
| Reduced Motion | 100% support | Must respect system settings | Test with system setting enabled |
| Flash Frequency Limit | Not exceeding 3Hz | Avoid photosensitive epilepsy risk | Review keyframe animations |

### Memory Constraints

| Constraint Type | Hard Limit | Explanation | Monitoring Method |
|---------|---------|------|---------|
| Animation Instance Count | Not exceeding 20 | Upper limit of retained animation objects | Memory panel heap snapshot |
| Memory Growth Rate | Not exceeding 5MB per minute | Long-running shouldn't continuously grow | Performance Monitor real-time monitoring |
| Cleanup Time Window | Within 1 second after component unmount | Animation resources must be released promptly | Check useEffect cleanup |
| Event Listeners | Remove promptly | Avoid event listener leaks | Chrome Memory Profiler |
| Timer Cleanup | 100% cleanup | All setTimeout/setInterval must be cleared | Code review + runtime detection |
| Off-screen Elements | Pause animations | Invisible elements shouldn't continue animating | Intersection Observer monitoring |

### Browser Compatibility Constraints

| Constraint Type | Minimum Support Version | Fallback Strategy | Detection Method |
|---------|------------|---------|---------|
| CSS Transform | IE 10+ | Hide animation if unsupported, show content directly | @supports query |
| CSS Animation | IE 10+ | Fallback to transition or static | Feature detection |
| Intersection Observer | Chrome 51+, Safari 12.1+ | Fallback to scroll event + throttling | Polyfill or feature detection |
| will-change | Chrome 36+, Firefox 36+ | Ignore if unsupported, rely on browser optimization | @supports query |
| backdrop-filter | Chrome 76+, Safari 9+ | Fallback to solid background | @supports query |
| prefers-reduced-motion | Chrome 74+, Safari 10.1+ | Provide simplified animation by default | matchMedia query |

### Development Constraint Rules

| Constraint Type | Mandatory Requirement | Explanation | Check Method |
|---------|---------|------|---------|
| Animation Configuration Centralization | Must be centrally managed | Not allowed to hard-code in components | Code review |
| Cleanup Functions | 100% implementation | Every animation must have cleanup logic | ESLint rules + Code Review |
| Performance Testing | Must pass | Must test on low-end devices before submission | CI process integration |
| Documentation Comments | Must add | Explain animation intent and parameter choices | Documentation coverage check |
| TypeScript Types | Must be complete | Animation parameters must have type definitions | tsc --noEmit check |
| Unit Testing | Cover trigger logic | Test animation trigger conditions not animations themselves | Jest coverage report |

## 8. Common Problem Diagnosis Table

### Performance Issues

| Symptom | Possible Cause | Solution | Priority |
|------|---------|---------|--------|
| Animation stuttering, frame rate below 30fps | Using reflow-triggering properties (left/top/width/height) | Use transform and opacity instead | P0 |
| Page jitter during scroll | Scroll listener triggers forced synchronous layout | Use Intersection Observer or requestAnimationFrame | P0 |
| First animation delay over 2 seconds | Animation library large size, slow first screen load | Use dynamic import or switch to CSS animation | P1 |
| Severe mobile device heating | Overuse of GPU acceleration or too many animations | Reduce simultaneous animated elements, lower complexity | P1 |
| Animation slows down in second half | JavaScript main thread blocked | Move compute-intensive tasks to Web Worker | P1 |
| Frame drops during page scroll | Animation recalculation triggered every scroll | Add throttling or use CSS scroll-driven animations | P1 |
| Slows down after prolonged use | Animation instances not cleaned up, memory leak | Clean up all animations and listeners on component unmount | P0 |
| Flashing when animation starts | will-change timing incorrect | Add will-change 100ms before animation, remove after | P2 |

### User Experience Issues

| Symptom | Possible Cause | Solution | Priority |
|------|---------|---------|--------|
| User feels slow operation response | Animation duration set too long (over 500ms) | Shorten to 200-300ms | P0 |
| Page switching feels sluggish | Transition animation time too long | Control to 300-500ms, remove unnecessary delays | P1 |
| User anxious during loading | No progress feedback, only infinite spinning | Add percentage progress or stage prompts | P1 |
| Animation chaotic with rapid clicks | Didn't handle animation interruption | Use animation library's interruption feature or reset state | P0 |
| User feedback of dizziness and discomfort | Didn't support prefers-reduced-motion | Detect system setting, provide static or simplified version | P0 |
| Animation feels abrupt | Using linear easing function | Change to ease-in-out or spring animation | P2 |
| Loading animation obscures content | Animation layer level too high and opaque | Lower z-index or use semi-transparent background | P1 |
| User thinks page is stuck | Animation over 3 seconds with no feedback | Add progress indicator or cancel button | P0 |

### Technical Integration Issues

| Symptom | Possible Cause | Solution | Priority |
|------|---------|---------|--------|
| Framer Motion animation not triggering | AnimatePresence used incorrectly or key not set properly | Ensure children have unique key, check mode setting | P1 |
| GSAP animation executes twice in React strict mode | useEffect called twice | Clean up animation instance in cleanup | P1 |
| Animation flashes during route switching | New page renders immediately, old page exit animation incomplete | Use AnimatePresence or wait for animation complete before switching | P1 |
| CSS animation doesn't work in Safari | Missing -webkit- prefix | Use Autoprefixer or manually add prefix | P0 |
| Animation errors during server-side rendering | Animation library attempts to access window object | Use dynamic import and disable ssr | P1 |
| TypeScript type errors | Animation library type definitions incomplete | Install @types package or add type declaration file | P2 |
| Next.js page transitions not smooth | App Router default behavior conflicts with animations | Use usePathname to monitor changes, manually control animations | P1 |
| Animation doesn't work after build | CSS class names compressed or removed | Check purgecss configuration, add safelist | P0 |

### Compatibility Issues

| Symptom | Possible Cause | Solution | Priority |
|------|---------|---------|--------|
| Animation jitter on iOS Safari | iOS rendering issues for certain CSS properties | Add -webkit-transform and use translateZ to force GPU acceleration | P1 |
| Animation not smooth in Firefox | Firefox's animation rendering strategy different | Adjust will-change usage or use JS animation library | P2 |
| Old browsers don't display animation | Don't support CSS Animation or Transform | Add @supports query, provide fallback plan | P1 |
| Animation has ghosting in Edge browser | Edge's composite layer optimization issue | Add backface-visibility: hidden | P2 |
| WeChat built-in browser stutters on Android | WeChat browser performance limitations | Simplify animation, reduce element count | P1 |
| Animation completely freezes on low-end devices | Insufficient device performance | Detect device performance, auto-downgrade to simplified animation | P0 |

### Debugging Issues

| Symptom | Possible Cause | Solution | Priority |
|------|---------|---------|--------|
| Cannot locate performance bottleneck | Not using correct debugging tools | Use Chrome DevTools Performance to record and analyze | P1 |
| Don't know which elements triggered reflow | Visualization tool not enabled | Use Rendering panel's Paint Flashing and Layout Shift | P1 |
| Cannot view composite layer situation | Not using Layers panel | Open Layers panel to view layer tree and reasons | P2 |
| Difficult to debug on mobile devices | Not using remote debugging | Use Chrome DevTools remote debugging or Eruda | P1 |
| Animation library internal errors hard to track | Source Map not loaded correctly | Check webpack devtool configuration | P2 |
| Uncertain if animation uses GPU | Haven't checked rendering layer info | View Composite events in Performance recording | P2 |

## 9. Output Format Requirements

### Animation Implementation Plan Output Template

```
# {Animation Name} Implementation Plan

## I. Requirements Analysis
- Animation Type: {transition/keyframes/interactive/scroll}
- Trigger Condition: {describe user operation or page event}
- Visual Effect: {detailed description of animation visual performance}
- User Goal: {information animation hopes to convey or behavior to guide}

## II. Technical Selection
- Selected Solution: {CSS/Framer Motion/GSAP/React Spring}
- Selection Reasons:
  - Performance Consideration: {why this solution has optimal performance}
  - Implementation Complexity: {development cost assessment}
  - Bundle Size Impact: {how many KB added}
  - Maintainability: {convenience for subsequent iteration}
- Alternative Solution: {replacement plan if main solution not feasible}

## III. Implementation Approach

### 3.1 Animation State Definition
- Initial State: {element's starting position, opacity, scale, etc.}
- Transition States: {intermediate keyframes during animation execution}
- End State: {final style after animation completion}

### 3.2 Animation Timing Design
- Total Duration: {X}ms
- Delayed Start: {Y}ms (if staggered effect)
- Easing Function: {ease-in/ease-out/spring(stiffness, damping)}
- Keyframe Distribution: {describe time points and states of keyframes}

### 3.3 Interaction Logic (if applicable)
- Trigger Method: {click/hover/scroll/drag}
- Gesture Recognition: {types of gestures needing recognition}
- Boundary Handling: {drag range limits or scroll boundaries}
- Interruption Mechanism: {how user operations interrupt animation}

## IV. Performance Optimization Strategy

### 4.1 Rendering Optimization
- Animation Properties: Only use {transform: translateX/translateY/scale/rotate, opacity}
- GPU Acceleration: {whether needed, how to enable}
- Layer Promotion: {which elements need independent composite layers}
- will-change Usage: {add before animation starts, remove after completion}

### 4.2 Resource Optimization
- On-demand Loading: {whether animation library dynamically imported}
- Code Splitting: {whether non-critical animations independent chunk}
- Lazy Execution: {whether off-screen elements pause animation}
- Memory Cleanup: {cleanup function implementation points}

### 4.3 Performance Monitoring
- Target Frame Rate: 60fps
- Performance Budget: Single frame not exceeding 16.67ms
- Monitoring Method: {which DevTools panels to use}
- Fallback Condition: {simplified plan for low-end devices}

## V. Accessibility Support

### 5.1 Reduced Motion Support
- Detection Method: prefers-reduced-motion media query
- Fallback Plan: {static display/simplified animation/opacity only}
- User Control: {whether to provide manual toggle}

### 5.2 Focus Management
- Focus Order: {animation doesn't affect keyboard navigation}
- Focus Visibility: {focus indicator clear during animation}
- Screen Reader: {semantic announcement of state changes}

## VI. Browser Compatibility

### 6.1 Support Scope
- Modern Browsers: {list tested passing versions}
- Mobile Browsers: {iOS Safari X+, Android Chrome X+}
- Fallback Support: {fallback plan for older browsers}

### 6.2 Compatibility Handling
- CSS Prefixes: {whether -webkit- and other prefixes needed}
- Polyfill: {list of needed polyfills}
- Feature Detection: {use @supports or JS detection}

## VII. Implementation Checklist

### 7.1 File Structure
- Animation Config File: {path and filename}
- Component Files: {location of animation-implementing components}
- Style Files: {CSS/SCSS file paths}
- Utility Functions: {location of helper functions}

### 7.2 Core Configuration
- Animation Duration Constants: {variable name defined in config file}
- Easing Function Definition: {parameters of custom easing curves}
- Responsive Breakpoints: {animation differences for different screen sizes}

### 7.3 Key Implementation Points
- Animation Trigger Logic: {how to listen and trigger}
- State Management: {how animation state stored and updated}
- Cleanup Logic: {useEffect cleanup or lifecycle methods}
- Error Handling: {fallback when animation fails}

## VIII. Testing Verification

### 8.1 Performance Testing
- [ ] Chrome DevTools Performance recording, frame rate stable at 60fps
- [ ] Mobile device testing, no obvious heating
- [ ] Low-end device throttling test, fallback plan works normally
- [ ] Memory leak detection, long-running no abnormalities

### 8.2 Functional Testing
- [ ] Animation executes correctly under all trigger conditions
- [ ] Rapid consecutive triggers don't become chaotic
- [ ] Animation can be correctly interrupted
- [ ] Boundary condition handling correct

### 8.3 Compatibility Testing
- [ ] Testing passed on Chrome/Firefox/Safari/Edge
- [ ] Testing passed on iOS and Android mobile browsers
- [ ] Old browser fallback plan verification
- [ ] Normal performance in accessibility mode

## IX. Future Optimization Space

- Possible Improvements: {list future optimizable directions}
- Performance Enhancement Space: {possibilities for further optimization}
- Functional Extension: {effects that can be expanded based on this animation}
- Code Refactoring: {more elegant implementation approaches}

## X. References

- Technical Documentation: {animation library official documentation links}
- Design Specifications: {Material Design/Apple HIG related chapters}
- Performance Guidelines: {Web.dev or MDN performance articles}
- Accessibility Standards: {relevant WCAG clauses}
```

### Performance Analysis Report Output Template

```
# {Project Name} Animation Performance Analysis Report

## Test Environment
- Browser: {Chrome version number}
- Device: {desktop/mobile device model}
- Screen Resolution: {width x height}
- CPU Throttling: {whether enabled, multiplier}
- Network Conditions: {Fast 3G/4G/WiFi}

## Performance Metrics

### Frame Rate Analysis
| Animation Scenario | Average FPS | Minimum FPS | Frame Drops | Pass |
|---------|---------|---------|---------|---------|
| Page Load Animation | X fps | Y fps | Z times | ✅/❌ |
| Scroll Parallax | X fps | Y fps | Z times | ✅/❌ |
| Modal Transition | X fps | Y fps | Z times | ✅/❌ |
| List Animation | X fps | Y fps | Z times | ✅/❌ |

### Performance Budget
| Metric | Budget Value | Actual Value | Over Budget |
|------|-------|-------|---------|
| Single Frame Max Time | 16.67ms | X ms | ✅/❌ |
| Main Thread Blocking Duration | < 50ms | X ms | ✅/❌ |
| GPU Layer Count | < 10 layers | X layers | ✅/❌ |
| JavaScript Execution Time | < 100ms | X ms | ✅/❌ |
| Memory Usage Growth | < 5MB/min | X MB/min | ✅/❌ |

## Problem Identification

### Performance Bottlenecks
1. {Problem description}
   - Location: {filename and line number}
   - Cause: {trigger reflow/heavy computation/memory leak}
   - Impact: {frame rate drop X fps, user experience Y}
   - Priority: P{0/1/2}

2. {Next problem...}

### Optimization Recommendations

#### High Priority (P0)
- [ ] {Optimization item 1}: {specific measures}, expected improvement {X} fps
- [ ] {Optimization item 2}: {specific measures}, expected reduction {Y} ms

#### Medium Priority (P1)
- [ ] {Optimization item 3}: {specific measures}, expected improvement {description}
- [ ] {Optimization item 4}: {specific measures}, expected reduction {Z} KB

#### Low Priority (P2)
- [ ] {Optimization item 5}: {specific measures}, code quality improvement

## Detailed Analysis

### Animation Rendering Pipeline
{Describe entire rendering flow from trigger to completion}
1. Trigger Phase: {JavaScript execution time}
2. Style Calculation: {Style time}
3. Layout Calculation: {Layout time}
4. Paint: {Paint time}
5. Composite: {Composite time}

### Key Findings
- Finding 1: {description}
- Finding 2: {description}
- Finding 3: {description}

## Before and After Optimization Comparison

| Metric | Before Optimization | After Optimization | Improvement Rate |
|------|-------|-------|---------|
| Average Frame Rate | X fps | Y fps | +Z% |
| First Animation Delay | X ms | Y ms | -Z% |
| Memory Usage | X MB | Y MB | -Z% |
| Bundle Size | X KB | Y KB | -Z% |

## Next Actions

### Immediate Execution (Within This Week)
- [ ] {Optimization item}: Owner {name}

### Planned Execution (Within This Month)
- [ ] {Optimization item}: Owner {name}

### Long-term Optimization (Next Quarter)
- [ ] {Optimization item}: Owner {name}

## Appendix

### Test Data Sources
- Performance recording files: {path}
- Lighthouse reports: {path}
- Screenshots and recordings: {path}

### Reference Benchmarks
- Industry standards: {links}
- Competitor comparison: {data}
```

### Animation Design Document Output Template

```
# {Feature Module} Animation Design Document

## Design Intent
{Describe animation design purpose, emotions to convey, user behaviors to guide}

## Animation Inventory

### Micro-interaction Animations
| Element | Trigger Condition | Animation Effect | Duration | Easing |
|------|---------|---------|------|------|
| Button | Hover | Background color transition + slight scale up | 200ms | ease-out |
| Button | Click | Scale down to 0.95x | 100ms | ease-in |
| Input Box | Focus | Border color transition + shadow expand | 200ms | ease-out |
| Checkbox | Check | Checkmark path animation + elastic scale | 300ms | spring |
| Switch | Toggle | Slider translate + background color transition | 250ms | ease-in-out |

### Page-level Animations
| Scenario | Enter Effect | Exit Effect | Duration | Technical Solution |
|------|---------|---------|------|---------|
| Page Switch | Slide in from right + fade in | Slide out left + fade out | 400ms | Framer Motion |
| Modal | Scale from center + fade in | Scale to center + fade out | 300ms | Framer Motion |
| Drawer | Slide in from side | Slide out to side | 350ms | CSS Transform |
| Dropdown Menu | Expand down + fade in | Collapse up + fade out | 200ms | CSS Animation |

### Loading and Feedback
| Scenario | Animation Type | Description | Duration | Trigger Condition |
|------|---------|------|---------|---------|
| First Screen Load | Skeleton screen shimmer | Left to right moving highlight gradient | 1.5s loop | Content not loaded |
| Button Loading | Spinner | Circular border rotation | 1s loop | Async operation in progress |
| Data Loading | Progress bar | Blue bar fills left to right | Based on progress | Long loading tasks |
| Operation Success | Checkmark path animation | Green checkmark drawing + scale | 500ms | Success callback |
| Operation Failure | Shake | Left-right sway | 400ms | Failure callback |

### Scroll-driven
| Scenario | Trigger Position | Animation Effect | Implementation |
|------|---------|---------|---------|
| Content Fade In | Element enters viewport bottom 80% | Fade in from 50px below | Intersection Observer |
| Parallax Background | Throughout scroll | Background moves at 0.5x speed | GSAP ScrollTrigger |
| Number Increment | Element enters viewport | Count from 0 to target value | React Spring |
| Progress Indicator | Throughout scroll | Top progress bar fills | CSS + Scroll Event |

## Design Specifications

### Duration Standards
- Micro-interactions (buttons, links): 200-300ms
- Small components (tooltips, dropdowns): 250-350ms
- Medium components (modals, drawers): 300-500ms
- Page transitions: 300-500ms
- Loading animations: 1-2 seconds per cycle

### Easing Functions
- ease-in: For element exit and collapse, slow start fast end
- ease-out: For element entrance and expand, fast start slow end
- ease-in-out: For back-and-forth transitions, slow both ends fast middle
- spring: For physics realism interactions, elastic effect

### Animation Hierarchy
- Z1: Background and bottom decorative animations, least prominent
- Z2: Content area animations, medium attention
- Z3: Interaction feedback and guidance animations, high attention

## Accessibility Design

### Reduced Motion Mode
- Fade animations: Keep opacity change, remove displacement and scale
- Slide animations: Change to simple fade in/out
- Spring animations: Change to linear or static switch
- Infinite loops: Stop or slow down speed

### Semantic Feedback
- Loading state: Provide aria-live="polite" announcement
- Success/failure: Provide role="alert" notification
- Progress change: Update aria-valuenow attribute

## Responsive Adaptation

### Mobile Adjustments
- Animation duration: Shorten by 20% (touch feedback needs faster response)
- Animation distance: Shorten by 30% (limited screen space)
- Gesture support: Swipe, long press, pinch and other mobile-specific interactions
- Performance protection: Lower complexity, reduce concurrent animation count

### Breakpoint Differences
- Small screen (< 640px): Simplify animations, keep only key transitions
- Medium screen (640-1024px): Standard animation effects
- Large screen (> 1024px): Can enhance details, smoother effects

## Brand Consistency
- Primary Easing: {brand-specific cubic-bezier curve}
- Signature Animation: {brand-recognizable specific animation effect}
- Color Transition: {gradient direction of brand color scheme}

## Implementation Priority
1. P0 (Must implement): {list core animations}
2. P1 (Important but not urgent): {list enhancement animations}
3. P2 (Nice to have): {list decorative animations}

## Acceptance Criteria
- [ ] All P0 animations implemented and pass performance testing
- [ ] All animations support prefers-reduced-motion
- [ ] Testing passed on mainstream browsers and mobile devices
- [ ] Design draft and implementation effect error less than 50ms
```

---

The above document follows structured, actionable principles, avoiding code examples, focusing on decision guidance, specification constraints, and problem diagnosis to help developers make correct animation implementation choices.

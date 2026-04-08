---
name: awesome-remotion
description: Best practices for Remotion video projects using React. Use when creating, scaffolding, or refactoring Remotion video projects with multiple scenes. Provides project structure conventions, scene composition patterns with Sequence+map, duration management best practices, and clean Root.tsx organization.
---

# Awesome Remotion

Best practices for creating multi-scene Remotion video projects.

## When to Use This Skill

Use this skill when the user wants to:
- Create a new Remotion video project from scratch
- Add scenes or refactor an existing Remotion project
- Organize a multi-scene video with proper structure
- Set up scene composition with animations

## Project Structure Convention

Follow this structure for all Remotion video projects:

```
src/
├── index.ts              # Entry point: registerRoot(RemotionRoot)
├── Root.tsx              # Composition definitions only (clean, no leftovers)
├── index.css             # Global styles (Inter font + monospace)
└── <VideoName>/          # Scene components directory (named after the video)
    ├── index.tsx         # Main video component (sequences all scenes)
    ├── Opening.tsx       # Scene 1 component
    ├── SceneTwo.tsx      # Scene 2 component
    ├── SceneThree.tsx    # Scene 3 component
    └── ...
```

Key principles:
- **Scene components** live together in one directory named after the video
- **Main video component** (`index.tsx`) is the orchestrator — it sequences all scenes
- **Root.tsx** contains ONLY `<Composition>` definitions — no leftover template code
- Each **scene component is self-contained** with its own animation logic

## Root.tsx Pattern

Root.tsx should be minimal and clean. Only define compositions here:

```tsx
// src/Root.tsx
import "./index.css";
import { Composition } from "remotion";
import { MyVideo, TOTAL_DURATION } from "./MyVideo";

export const RemotionRoot: React.FC = () => {
  return (
    <>
      <Composition
        id="MyVideo"
        component={MyVideo}
        durationInFrames={TOTAL_DURATION}
        fps={30}
        width={1920}
        height={1080}
      />
    </>
  );
};
```

Rules for Root.tsx:
- Import main video component and its exported `TOTAL_DURATION`
- One composition per video (the full video)
- Do NOT include leftover HelloWorld / Logo template compositions
- Do NOT include individual scene compositions unless explicitly needed for debugging

## Main Video Component Pattern (index.tsx inside scene dir)

The main video component uses **array-based `<Sequence>` mapping** for clean scene orchestration:

```tsx
// src/MyVideo/index.tsx
import { AbsoluteFill, Sequence } from "remotion";
import { Opening } from "./Opening";
import { SceneTwo } from "./SceneTwo";
import { SceneThree } from "./SceneThree";

const FPS = 30;

// Centralized duration config - single source of truth
const SCENES = [
  // [sceneName,   Component,    startFrame,   durationInFrames]
  ["Opening",     Opening,      0,            5 * FPS],     // 0s - 5s
  ["SceneTwo",    SceneTwo,     5 * FPS,      7 * FPS],     // 5s - 12s
  ["SceneThree",  SceneThree,   12 * FPS,     10 * FPS],    // 12s - 22s
] as const;

// Export total duration for use in Root.tsx Composition
export const TOTAL_DURATION = SCENES.reduce((sum, [, , , dur]) => sum + dur, 0);

export const MyVideo: React.FC = () => {
  return (
    <AbsoluteFill style={{ backgroundColor: "#0f172a" }}>
      {SCENES.map(([name, Component, from, duration]) => (
        <Sequence key={name} from={from} durationInFrames={duration}>
          <Component />
        </Sequence>
      ))}
    </AbsoluteFill>
  );
};
```

Why this pattern is preferred:
- **Array-based `.map()`** is cleaner than hand-writing each `<Sequence>` with manual addition
- **`SCENES` array** is the single source of truth for timing
- **`TOTAL_DURATION` export** prevents mismatched numbers between component and composition
- **Adding/removing/reordering scenes** = just editing one array row
- **Each scene's internal frame count starts from 0** (Sequence handles offset)

## Scene Component Pattern

Each scene component is a standalone React component focused on its own content and animations:

```tsx
// src/MyVideo/Opening.tsx
import {
  AbsoluteFill,
  useCurrentFrame,
  useVideoConfig,
  interpolate,
  spring,
} from "remotion";

export const Opening: React.FC = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  // All animations reference frame starting from 0 within this scene
  const fadeIn = interpolate(frame, [0, 20], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });

  // ... more animations ...

  return (
    <AbsoluteFill style={{ backgroundColor: "#0f172a" }}>
      {/* Scene content */}
    </AbsoluteFill>
  );
};
```

Key rules for scene components:
- Always use `useCurrentFrame()` — frame 0 = start of this scene (thanks to Sequence)
- Always wrap content in `<AbsoluteFill>`
- Set background color in the AbsoluteFill (ensures no transparency gaps)
- Keep each scene self-contained — don't reference other scenes' state
- Use `spring()` for natural motion, `interpolate()` for linear/timed transitions

## Animation Cheat Sheet

Common animation patterns used across scenes:

### Slide In (from left/right)
```tsx
const slideProgress = spring({ frame: frame - delay, fps, config: { damping: 12 } });
const xPosition = interpolate(slideProgress, [0, 1], [-500, 0]); // from left
```

### Scale Pop-In
```tsx
const scaleProgress = spring({ frame: frame - delay, fps, config: { damping: 8 } };
// Apply via transform: `scale(${scaleProgress})`
```

### Fade In
```tsx
const opacity = interpolate(frame, [startFrame, endFrame], [0, 1], {
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

### Staggered Text Lines
```tsx
// Animate lines appearing one after another
const line1Opacity = interpolate(frame, [10, 30], [0, 1], { clamp: true });
const line2Opacity = interpolate(frame, [35, 55], [0, 1], { clamp: true });
```

## Global CSS Template

```css
/* src/index.css */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&family=Fira+Code:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500;600;700&display=swap');

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'SF Pro Display', sans-serif;
  background-color: #0f172a;
  color: #f8fafc;
}

code, pre, .monospace {
  font-family: 'Fira Code', 'JetBrains Mono', 'Cascadia Code', monospace;
}
```

## Common Color Palette (Dark Theme Videos)

| Token | Value | Usage |
|-------|-------|-------|
| Background | `#0f172a` | Main background |
| Surface | `#1e293b` | Cards, panels |
| Border | `#334155` | Panel borders |
| Primary text | `#f8fafc` | Headings |
| Secondary text | `#94a3b8` | Body text |
| Muted text | `#64748b` | Hints, labels |
| Accent purple | `#c084fc` | @ scope, namespaces |
| Accent yellow | `#fbbf24` | Highlights, warnings |
| Accent pink | `#f472b6` | Version tags |
| Accent green | `#4ade80` | Commands, success |
| Accent red | `#CB3837` | npm brand |
| Python blue | `#387EB8` | Python branding |

## Workflow: Creating a New Video

1. Create scene directory under `src/<VideoName>/`
2. Write individual scene components (`.tsx`)
3. Create `src/<VideoName>/index.tsx` as the orchestrator with `SCENES` array + `TOTAL_DURATION` export
4. Update `src/Root.tsx` to add the new `<Composition>` pointing to the main video component
5. Run `npm run dev` to preview in Studio
6. Click the composition in Studio sidebar to preview full video timeline

## Rendering

To render the final video:

```bash
# Via Studio UI: Click "Render" button on the composition
# Or via CLI:
npx remotion render src/index.ts MyVideo out/video.mp4
```

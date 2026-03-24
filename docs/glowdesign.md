# GlowDesign Editor

GlowDesign Lite is NoverFly's built-in **visual website editor** — a drag-and-drop builder that lets you create professional, responsive websites without writing code.

---

## Overview

GlowDesign provides:
- 🎨 Visual drag-and-drop interface
- 📐 Precise layout control (Flexbox, Grid, absolute positioning)
- 🎯 Component-based architecture
- 📱 Responsive breakpoints
- ⚡ Real-time preview
- 🧩 Reusable component library
- 🎬 Animations and interactions
- 🔗 CMS data binding
- 💾 Auto-save and version history

---

## Interface Layout

```
┌──────────────────────────────────────────────────────────┐
│  [≡] Logo   Page: Home ▼   [↶ ↷]   [👁 Preview] [Publish]│
├──────────┬──────────────────────────────┬────────────────┤
│          │  ┌─────────────────────────┐ │                │
│  PANELS  │  │                         │ │  PROPERTIES    │
│          │  │       CANVAS            │ │                │
│  Pages   │  │                         │ │  Style         │
│  Layers  │  │    Your website         │ │  - Typography  │
│  Comps   │  │    renders here         │ │  - Colors      │
│  Assets  │  │                         │ │  - Spacing     │
│  CMS     │  │                         │ │  - Layout      │
│          │  │                         │ │  - Size        │
│          │  └─────────────────────────┘ │  - Position    │
│          │  [🖥 Desktop] [💻 Tablet] [📱]│  - Effects     │
└──────────┴──────────────────────────────┴────────────────┘
```

---

## Components

### Layout Components

| Component | Description |
|-----------|-------------|
| **Section** | Full-width container, typically a page section |
| **Container** | Centered content wrapper with max-width |
| **Div Block** | Generic container for grouping elements |
| **Grid** | CSS Grid layout with rows and columns |
| **Columns** | Pre-built column layouts (2, 3, 4 columns) |

### Content Components

| Component | Description |
|-----------|-------------|
| **Heading** | H1–H6 text headings |
| **Paragraph** | Body text block |
| **Text Link** | Inline or block link |
| **Image** | Responsive image with lazy loading |
| **Video** | Embedded video player (YouTube, Vimeo, self-hosted) |
| **Icon** | SVG icon from built-in library |
| **Rich Text** | WYSIWYG editable content block |
| **Embed** | Custom HTML/CSS/JS embed |

### Form Components

| Component | Description |
|-----------|-------------|
| **Form** | Form wrapper with submit action |
| **Input** | Text, email, password, number, tel |
| **Textarea** | Multi-line text input |
| **Select** | Dropdown menu |
| **Checkbox** | Single checkbox |
| **Radio** | Radio button group |
| **File Upload** | File input with drag-and-drop |
| **Button** | Submit or action button |

### Interactive Components

| Component | Description |
|-----------|-------------|
| **Navbar** | Responsive navigation bar |
| **Tabs** | Tabbed content panels |
| **Accordion** | Collapsible content sections |
| **Modal** | Popup dialog/overlay |
| **Slider** | Image or content carousel |
| **Dropdown** | Dropdown menu with trigger |
| **Tooltip** | Hover tooltip |

### E-Commerce Components

| Component | Description |
|-----------|-------------|
| **Product Card** | Display a product with image, title, price |
| **Product Gallery** | Image gallery for product pages |
| **Add to Cart** | Button to add items to cart |
| **Cart** | Shopping cart summary |
| **Checkout Form** | Complete checkout flow |

---

## Styling

### Typography

Set font properties for any text element:
- **Font Family** — Google Fonts, system fonts, or custom uploads
- **Font Size** — px, rem, em, vw
- **Font Weight** — 100 to 900
- **Line Height** — unitless or fixed
- **Letter Spacing** — px or em
- **Text Transform** — uppercase, lowercase, capitalize
- **Text Align** — left, center, right, justify

### Colors

- **Text Color** — Any hex, RGB, or HSL value
- **Background Color** — Solid or gradient
- **Border Color** — Per-side control
- **Opacity** — 0 to 100%

### Spacing

- **Margin** — Outside spacing (top, right, bottom, left)
- **Padding** — Inside spacing (top, right, bottom, left)
- Individual or shorthand values

### Layout

- **Display** — Block, Flex, Grid, Inline, None
- **Flex Direction** — Row, Column
- **Justify Content** — Start, Center, End, Space Between, Space Around
- **Align Items** — Start, Center, End, Stretch
- **Gap** — Spacing between flex/grid children

### Size

- **Width** — px, %, vw, auto
- **Height** — px, %, vh, auto
- **Min/Max Width**
- **Min/Max Height**
- **Overflow** — Visible, Hidden, Scroll, Auto

### Position

- **Static** — Normal document flow
- **Relative** — Offset from normal position
- **Absolute** — Relative to nearest positioned ancestor
- **Fixed** — Relative to viewport
- **Sticky** — Toggles between relative and fixed

### Effects

- **Box Shadow** — offset, blur, spread, color
- **Border Radius** — per-corner control
- **Filters** — blur, brightness, contrast, grayscale
- **Backdrop Filter** — glass effects
- **Transitions** — property, duration, easing

---

## Responsive Design

GlowDesign supports three breakpoints:

| Breakpoint | Min Width | Icon |
|-----------|-----------|------|
| Desktop | 1280px | 🖥️ |
| Tablet | 768px | 💻 |
| Mobile | 375px | 📱 |

### How Cascading Works

1. **Desktop styles apply everywhere** by default
2. **Tablet styles override desktop** for screens ≤ 1279px
3. **Mobile styles override tablet** for screens ≤ 767px

### Tips:
- Design desktop first, then adjust for tablet and mobile
- Use Flexbox with `flex-wrap` for automatic responsive layouts
- Set images to `width: 100%` and `height: auto` for responsive images
- Use `vw` units for fluid typography

---

## Pages

### Creating Pages
1. Open the **Pages** panel (left sidebar)
2. Click **+ New Page**
3. Enter page name and URL slug
4. The page is created and opens in the canvas

### Page Settings
- **Title** — Browser tab title (SEO)
- **Description** — Meta description (SEO)
- **Slug** — URL path (e.g., `/about`, `/contact`)
- **Open Graph** — Social media preview image and text
- **Custom Code** — Head and body code injection

---

## Layers Panel

The layers panel shows the DOM tree of your page:

```
▼ Body
  ▼ Section (Hero)
    ▼ Container
      ▶ Heading "Welcome"
      ▶ Paragraph
      ▶ Button "Get Started"
  ▼ Section (Features)
    ▼ Grid (3 cols)
      ▶ Feature Card 1
      ▶ Feature Card 2
      ▶ Feature Card 3
```

- Click to select elements
- Drag to reorder
- Right-click for context menu (duplicate, delete, wrap, etc.)

---

## Assets

Upload and manage media files:

- **Images** — JPG, PNG, WebP, SVG, GIF
- **Videos** — MP4, WebM
- **Documents** — PDF, DOC
- **Fonts** — TTF, WOFF, WOFF2

All assets are stored in Flivex Cloud Storage with CDN delivery.

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl + S` | Save |
| `Ctrl + Z` | Undo |
| `Ctrl + Shift + Z` | Redo |
| `Ctrl + C` | Copy element |
| `Ctrl + V` | Paste element |
| `Ctrl + D` | Duplicate element |
| `Delete` | Delete selected element |
| `Ctrl + G` | Group elements |
| `Ctrl + Shift + G` | Ungroup |
| `Ctrl + [` | Send backward |
| `Ctrl + ]` | Bring forward |

---

## Next Steps

- [CMS & Database](cms.md) — Add dynamic content
- [E-Commerce](ecommerce.md) — Build your store
- [Deployment](deployment.md) — Publish your site

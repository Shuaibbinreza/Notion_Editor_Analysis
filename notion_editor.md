## 📝 **Notion Rich Text Editor – Features Overview**

---

### 🔤 **Text Formatting**
- **Bold** (`Cmd/Ctrl + B`)
- **Italic** (`Cmd/Ctrl + I`)
- **Underline**
- **Strikethrough**
- **Inline Code** (`Cmd/Ctrl + E`)
- **Highlighting** (various colors)
- **Text Color** customization

---

### 🔗 **Links & Mentions**
- **Hyperlinks**: Add links to text
- **@Mentions**: 
  - Pages
  - People
  - Dates (`@today`, `@tomorrow`, or specific dates)

---

### 🧱 **Content Blocks**
Each of these is its own movable block:
- **Paragraph**
- **Headings**:
  - Heading 1
  - Heading 2
  - Heading 3
- **Bullet list**
- **Numbered list**
- **To-do list / Checkbox**
- **Toggle list** (collapsible)
- **Quote**
- **Callout** (highlighted note with emoji)
- **Divider** (horizontal line)

---

### 🧑‍💻 **Code & Math**
- **Inline code**
- **Code block** (multi-line, with language highlighting)
- **Math equations** (LaTeX)

---

### 🖼️ **Media Embeds**
- **Image**
- **Video**
- **Audio**
- **PDF**
- **File Upload**
- **Web Bookmark preview**
- **Embed external tools** (Figma, Google Maps, Miro, Loom, etc.)

---

### 🗃️ **Database Views**
- **Inline Databases**:
  - Table
  - Board (Kanban)
  - Gallery
  - Calendar
  - Timeline
  - List
- These support rich text fields too

---

### 🔁 **Drag-and-Drop**
- Reorder blocks with drag handles
- Nest blocks inside toggle lists, quotes, callouts, etc.

---

### ⚙️ **Keyboard & Markdown Shortcuts**
- `/` command menu for block creation
- Markdown support:
  - `# Heading`, `## Heading`, `### Heading`
  - `*` or `-` for bullet list
  - `1.` for numbered list
  - `> Quote`
  - `` `inline code` ``

---

### 🧠 **Other Features**
- **Synced blocks** (content mirrored across pages)
- **Templates**
- **Breadcrumb navigation**
- **Full-page or inline pages**
- **Comments on text**
- **Undo/redo & version history**

---

## 🛠️ Notion Rich Text Editor: How CRUD Works (Step-by-Step)

---

### ✍️ **1. You Start Typing in the Editor**

**What Happens Internally:**
- Notion captures **each keystroke** in the frontend.
- It updates a **local in-memory state** that represents the current block’s content (a JSON-like object).
- No network request is sent immediately (for performance reasons).

**Key Concepts:**
- **Virtual document model** (block tree in memory)
- Debounced or batched saves

---

### 💾 **2. Notion Auto-Saves Your Input (Create/Update)**

After you pause typing (usually ~500ms or 1s):

**Frontend:**
- Constructs an **update operation** (e.g., modify a block’s `rich_text`)
- Sends a request to the backend API like:

```http
PATCH /v1/blocks/<block_id>
Content-Type: application/json

{
  "paragraph": {
    "rich_text": [
      {
        "type": "text",
        "text": { "content": "Hello, Notion!" },
        "annotations": { "bold": false }
      }
    ]
  }
}
```

**Backend:**
- Parses the block update
- Validates structure and permissions
- Saves to the database (PostgreSQL or internal custom store)
- Updates version history
- Sends real-time changes to collaborators (via WebSocket)

---

### 📥 **3. Loading a Page (Read)**

When you open a Notion page:

**Frontend sends:**
```http
GET /v1/pages/<page_id>
```

Backend responds with:
- A full **block tree** of the page:
```json
{
  "results": [
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": {
        "rich_text": [...]
      }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [...]
      }
    }
  ]
}
```

**Frontend renders** this block structure into interactive React components.

---

### 🗑️ **4. Deleting Content**

When you delete:
- A block → deletes the block object
- Some text → updates the block's `rich_text` content

**Request example (delete block):**
```http
DELETE /v1/blocks/<block_id>
```

If it’s just removing text:
```http
PATCH /v1/blocks/<block_id>
{
  "paragraph": {
    "rich_text": []
  }
}
```

---

### ➕ **5. Creating New Blocks**

You hit `Enter` and a new paragraph block is created.

**Frontend sends:**
```http
POST /v1/blocks
{
  "parent": { "type": "page_id", "page_id": "<page_id>" },
  "type": "paragraph",
  "paragraph": {
    "rich_text": [
      {
        "type": "text",
        "text": { "content": "New line" }
      }
    ]
  }
}
```

The backend returns a block ID, and the new block is inserted in the frontend’s state and DOM.

---

### 🔁 **6. Real-Time Collaboration**

Notion uses **WebSockets** to:
- Broadcast block-level updates to other users in real time
- Sync cursor positions, selection highlights, and live edits
- Resolve conflicts using CRDTs (likely)

---

## 🧠 Summary: Notion Rich Text CRUD Flow

| Action         | Frontend           | Backend                            |
|----------------|--------------------|-------------------------------------|
| Type Text      | Updates local state | Nothing (yet)                       |
| Auto-Save      | Sends PATCH         | Updates block `rich_text`           |
| Open Page      | Sends GET           | Returns block tree                  |
| Create Block   | Sends POST          | Adds new block to document tree     |
| Update Block   | Sends PATCH         | Modifies existing block             |
| Delete Block   | Sends DELETE        | Marks block as deleted              |
| Real-Time Sync | WebSocket messages  | Broadcasts ops to collaborators     |

---

## 🧱 **Block Format in Notion (Storage Format)**

Each block is saved as a **JSON object**, typically with this structure:

```json
{
  "object": "block",
  "id": "block-id-123",
  "type": "paragraph",                // or "heading_1", "to_do", "bulleted_list_item", etc.
  "paragraph": {
    "rich_text": [
      {
        "type": "text",
        "text": { "content": "This is a paragraph", "link": null },
        "annotations": {
          "bold": false,
          "italic": false,
          "underline": false,
          "strikethrough": false,
          "code": false,
          "color": "default"
        },
        "plain_text": "This is a paragraph",
        "href": null
      }
    ]
  },
  "has_children": false,
  "parent": {
    "type": "page_id",
    "page_id": "page-id-456"
  }
}
```

### 🔁 **Yes, Blocks Are Organized**
Blocks are stored **in a tree-like structure**, and yes — the order and nesting are maintained.

---

## 📂 **Block Tree Structure (Organization)**

1. **Top-Level Structure**
   - Each page has a `page_id`
   - That page has a list of **child blocks**, ordered like a list

2. **Nested Blocks**
   - Some blocks (like toggle, list, quote) can have **children blocks**
   - These children are stored recursively under that block

```json
{
  "object": "block",
  "type": "toggle",
  "toggle": {
    "rich_text": [{ "text": { "content": "Click me" } }]
  },
  "has_children": true,
  "children": [
    {
      "type": "paragraph",
      "paragraph": {
        "rich_text": [{ "text": { "content": "I was hidden!" } }]
      }
    }
  ]
}
```

---

## 📦 **Storage & Retrieval Organization**

### ✅ On Save:
- Each block is saved individually with:
  - Its unique ID
  - Type and content
  - Reference to its parent (like a page or another block)
  - Its order/index in the parent
  - Children (if any)

### 📥 On Read:
- When loading a page:
  - A `GET` request fetches the **parent page**
  - Then a **`GET children` API** call fetches **all blocks** in order
  - If a block has `has_children: true`, Notion will:
    - Either fetch them inline
    - Or lazily load the children when needed

---

## 📌 Summary

| Aspect             | How It Works                                                                                                                                  |
|--------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| **Format**         | JSON block objects (not HTML/Markdown)                                                                                                       |
| **Block Types**    | `paragraph`, `heading_1`, `to_do`, `toggle`, `list_item`, `code`, `embed`, etc.                                                              |
| **Text Handling**  | `rich_text` arrays with styling annotations (bold, italic, etc.)                                                                             |
| **Hierarchy**      | Blocks are structured as a **tree**, each with parent/child relationships                                                                    |
| **Storage**        | Each block saved individually with type, ID, parent, order, and children if applicable                                                       |
| **Retrieval**      | Blocks are retrieved **in order**, and nested blocks are either returned inline or fetched recursively                                       |

---

## ❌ Can We Use Notion's Actual Editor in Your Web Page?

**No**, Notion's editor is **not open source**, and they don't provide an embeddable or SDK version of their editor. It's a proprietary system deeply integrated with their backend and block-based architecture.

---

## ✅ But... We *Can* Use Open-Source Alternatives That Work Like Notion

There are several **open-source Notion-style editors** that you *can* embed and extend:

---

### 🔥 1. **Tiptap**
**Tech:** Vue or React  
**Inspired by:** ProseMirror (powerful structured rich-text engine)  
**Best For:** Full Notion-like experience (blocks, markdown shortcuts, nesting)

👉 [https://tiptap.dev](https://tiptap.dev)

```bash
# Install Tiptap for React
npm install @tiptap/react @tiptap/starter-kit
```

> We can define block nodes like `paragraph`, `heading`, `taskList`, `codeBlock`, and even add `/` commands.

---

### 🧠 2. **Remirror**
**Tech:** React  
**Based on:** ProseMirror  
**Highlights:** Structured editing, collaborative features, markdown support

👉 [https://remirror.io](https://remirror.io)

---

### ⚙️ 3. **Lexical (by Meta)**
**Tech:** React  
**Minimal, lightweight**, but powerful  
**Use Cases:** Notion, Messenger-style editors

👉 [https://lexical.dev](https://lexical.dev/)

> It's modular and performant — you can add custom nodes for things like embeds, tables, checklists, etc.

---

### 🧱 4. **BlockNote**
**Tech:** React  
**Built on top of:** Tiptap  
**Out of the box:** Notion-style block editing, markdown support, collaborative editing (coming soon)

👉 [https://blocknotejs.org](https://blocknotejs.org/)

---

## ✅ So What We Should Do:
We can **replace TinyMCE** with:
- 🔄 A **block-based** editor like **Tiptap** or **BlockNote**
- And optionally save the data in **JSON format** (just like Notion does)
- Render it back by parsing the JSON

---

## 🔄 Key Differences Between TinyMCE vs. Notion-like Editor

| Feature                  | TinyMCE                            | Notion-style Editor (e.g. Tiptap/BlockNote)                       |
|--------------------------|-------------------------------------|------------------------------------------------------------------|
| **Content Model**        | HTML string                         | JSON block tree                                                  |
| **Editing Style**        | WYSIWYG (like Word)                 | Block-based (like Notion or Roam)                                |
| **Storage Format**       | Raw HTML                            | Structured JSON                                                  |
| **Rendering**            | Directly use `innerHTML`            | Need custom renderers (React/Vue components)                     |
| **Plugins**              | Many built-in plugins               | Extend with custom nodes or extensions                           |
| **Paste Handling**       | Auto-paste as HTML                  | Custom logic for converting HTML → blocks                        |
| **Content Parsing**      | HTML parsers available              | Need block serializer/deserializer                               |
| **Collaboration**        | Basic                               | Easy to add CRDT-based collab                                    |
| **Styling Options**      | Classic toolbar buttons             | Markdown shortcuts + block actions (`/` menu etc.)               |

---

## 🚧 Main Challenges You’ll Face

### 1. **Storage Format Transition**
**TinyMCE** stores content as HTML (`<p>Hello</p>`), while **Notion-style editors** use JSON:

```json
{
  "type": "paragraph",
  "content": [{ "type": "text", "text": "Hello" }]
}
```

**Challenge:** You’ll need a **migration strategy** for converting old HTML to JSON block data — either with:
- Custom converter
- Tiptap/ProseMirror HTML → JSON tools

---

### 2. **Rendering Content**
With TinyMCE, you just dump saved HTML into a `div`.

With a Notion-style editor, you **need a renderer** that reads JSON and converts to actual DOM nodes.

**Solution:** Use the same library (like Tiptap) to **render content** when not editing.

---

### 3. **Block-Based Thinking**
Notion-like editors enforce that **each block is self-contained**:
- You can’t just slap in raw `<div>`s or nested HTML
- You have to **define custom blocks** for things like embeds, collapsible sections, equations

---

### 4. **Customization**
TinyMCE offers a **plug-and-play editor**, Notion-style editors require:
- Building or customizing the toolbar
- Handling block types manually
- Integrating features like `/` commands, drag/drop, etc.

> More flexible, but **more work** up front.

---

### 5. **Form Submissions / Backend Handling**
If our backend expects:
- `HTML` → works with TinyMCE directly
- `JSON` → we'll need to **update our APIs** to accept block JSON, and possibly transform it to HTML if needed for emails or PDF generation

---

### 6. **Markdown and Shortcut Differences**
TinyMCE is toolbar-first.
Notion-like editors focus on:
- `/` commands
- Markdown-like shortcuts (`#` → heading, `-` → bullet)

This might require **retraining users** or tweaking UX.

---

## ✅ When Notion-style Is Worth It

- We need **modular block editing** (text, image, code, embeds as separate components)
- We care about **structured data** instead of raw HTML
- We plan to build **collaborative**, modern editing experiences (e.g. Notion, Linear, Craft)

---

## What about pdf conversion?
Since we're **saving HTML from TinyMCE and converting it to PDF**, and now we're thinking of switching to a **Notion-like (block-based) editor**, the key challenge is this:

> **Notion-style editors don’t output HTML directly. They use a block-based JSON structure.**

So now the big question is:  
👉 **How do we go from JSON blocks → PDF?**

---

## ✅ Our Options for Converting Notion-style JSON to PDF

### 🛠 Option 1: **Convert JSON blocks → HTML → PDF**

1. **Use the editor's renderer (like Tiptap) to convert block JSON to HTML**
2. Save that HTML in our DB
3. Use our existing HTML → PDF logic (like wkhtmltopdf or dompdf)

#### Example Flow:
```jsx
// Get editor content (Tiptap or BlockNote)
const editorJson = editor.getJSON();

// Convert JSON → HTML (custom function or built-in renderer)
const html = renderToHTML(editorJson); // We write or use a helper

// Send to backend for PDF generation
```

This is the **most flexible** approach.

---

### 🧩 Option 2: **Server-side rendering of blocks**

If we want to generate the PDF on the backend (e.g., using PHP/Laravel or Node), we’ll:

1. Send the block JSON to the backend
2. Create a custom renderer that turns JSON into styled HTML (or use a JS library if we're using Node)
3. Feed that HTML into our PDF tool

---

### 🖼 Option 3: **Render DOM and Print to PDF**

If our site already renders the editor content in a "read mode" (like a Notion page), we can:

1. Load the block content into a display view
2. Use JS libraries like `html2pdf.js`, `jsPDF`, or browser `window.print()` to export the visible DOM as PDF

✅ Good for quick solutions  
❌ Less control over styles and PDF layout

---

## 📦 Bonus: Tiptap/BlockNote → HTML Tools

- **Tiptap**: We can use `generateHTML` to convert the JSON to clean HTML:

```ts
import { generateHTML } from '@tiptap/html'
const html = generateHTML(editorJson, extensions)
```

- **BlockNote**: Exposes similar serialization or we can map over blocks manually

---

## 🔁 Summary

| Goal                        | Solution                                                                 |
|-----------------------------|--------------------------------------------------------------------------|
| Save content                | Save JSON (from editor.getJSON())                                        |
| Convert to HTML             | Use `generateHTML()` or custom renderer                                  |
| Convert to PDF              | Use our existing HTML → PDF pipeline (wkhtmltopdf, dompdf, etc.)        |

---

## 🛠️ Using a Notion-like Editor with CakePHP (and jQuery)
Since we’re working with **CakePHP** and **jQuery**, and not using React, integrating Tiptap or BlockNote directly might be a bit tricky. However, there are still ways to use **block-based editing** in our CakePHP project. Let’s break it down into actionable steps:

---

### 1. **Use a Vanilla JavaScript Notion-like Editor (e.g., Tiptap JS)**

While Tiptap and BlockNote are built for React, we can still use their **JavaScript versions** (they provide plain JavaScript implementations). Here's how:

#### Option A: **Tiptap (Plain JS)**

Tiptap offers **vanilla JavaScript** support (without React). We can use the same **block-based editing** but with standard DOM elements and JavaScript.

- **Steps to Integrate:**
  1. **Install Tiptap and Dependencies**:
     - Install Tiptap via npm or include it via CDN.
     - Tiptap has a simple integration for **vanilla JS** that doesn’t require React.
     
     ```bash
     npm install @tiptap/core @tiptap/starter-kit
     ```

  2. **Set Up Editor (Vanilla JS)**:
     Here's a basic Tiptap setup for a CakePHP app:

     ```html
     <div id="editor"></div>

     <script src="path/to/tiptap.js"></script>
     <script>
       const { Editor } = window.Tiptap;

       const editor = new Editor({
         element: document.querySelector('#editor'),
         extensions: [
           // Add your extensions here (e.g., paragraph, heading, bullet lists)
           new window.Tiptap.StarterKit(),
         ],
         content: "<p>Type your content here</p>", // Initial content
       });

       // Save the content as JSON (to send to your CakePHP backend)
       const jsonContent = editor.getJSON();
       console.log(jsonContent); // Send this to your backend
     </script>
     ```

  3. **Save JSON to CakePHP Backend**:
     - Send the **JSON block** to your CakePHP backend using **AJAX** (via jQuery).
     - Save this in your database as a JSON object.

     ```javascript
     $.ajax({
       type: 'POST',
       url: '/your-cakephp-controller/saveContent',
       data: { content: JSON.stringify(jsonContent) },
       success: function(response) {
         console.log("Content saved successfully");
       },
       error: function(err) {
         console.log("Error saving content");
       }
     });
     ```

#### Option B: **BlockNote (Vanilla JS)**

- BlockNote also has a **vanilla JS** version that you can embed directly into your page.

- **Steps**:
  1. **Include BlockNote** in your project (either via CDN or npm).
  2. Use the **block-based editor** to allow content editing.
  3. Similar to Tiptap, you’d send the **JSON blocks** to your CakePHP backend using jQuery AJAX.

---

### 2. **Convert JSON to HTML for PDF Generation**
Once you have the JSON data from the Notion-like editor (Tiptap or BlockNote), you’ll need to convert that JSON to HTML (which is needed for PDF generation).

- **Using Tiptap's JS Renderer**:
  Tiptap provides a `generateHTML` function to convert the JSON content into HTML:
  
  ```javascript
  const html = generateHTML(jsonContent, [
    // Define extensions to help with HTML conversion
    new window.Tiptap.StarterKit(),
  ]);
  ```

  Once you have the HTML, you can send it to the backend (CakePHP) and use your existing **wkhtmltopdf** or **dompdf** pipeline to generate a PDF.

---

### 3. **Backend (CakePHP) Handling**
In CakePHP, we'll need to:
1. **Receive the JSON content** via POST.
2. **Store it** as JSON in your database.
3. **Convert the JSON to HTML** when you need to render the page and pass it to PDF tools.
4. **Generate PDF** using your existing workflow (using wkhtmltopdf, dompdf, etc.).

For example, in CakePHP:
```php
public function saveContent() {
    $this->autoRender = false;
    $data = $this->request->getData('content'); // JSON block content
    $decodedContent = json_decode($data, true);

    // Save JSON to the database
    $this->loadModel('Contents');
    $contentEntity = $this->Contents->newEntity();
    $contentEntity->content = json_encode($decodedContent);
    $this->Contents->save($contentEntity);

    $this->response->withStatus(200)->send();
}
```

For generating HTML from JSON and sending it to **wkhtmltopdf**:
```php
// Convert JSON to HTML (you'll need to implement or use a parser for the block content)
$htmlContent = convertJsonToHtml($decodedContent);

// Use wkhtmltopdf to generate PDF
exec("wkhtmltopdf " . escapeshellarg($htmlContent) . " output.pdf");
```

---

 # CRDTs for Collaborative Editors


By JJ Salley

---

## Welcome

This talk will cover **CRDTs**, or Conflict-Free Replicated Data Types.\
They sound scary, but don’t worry—we’ll break them down clearly.

---

## Prep for CRDTs: Why Do We Care?

### Real-World Problem:

You're editing a document with others, but you all lose internet. What happens?

- Most systems crash, lock up, or overwrite each other.
- **We want systems that work offline, then merge smoothly when reconnected.**

This is what CRDTs make possible.

---

## Theory of CRDTs (Gently Introduced)

**CRDT** = Conflict-Free Replicated Data Type\
That’s a fancy way of saying:

> "A shared document you can edit at the same time as someone else without breaking it."

---

## What Makes CRDTs Special?

### The Naive Way: Shared Array of Bytes

- Think: a text file shared through Dropbox
- If two people change it at once: 💥 Conflict!

### The CRDT Way

- Each edit is recorded **with intent**
- Even if edits happen at the same time, they merge safely and consistently

---

## Metaphor Time

Imagine you’re playing LEGO with a friend:

- You each add blocks without seeing each other
- Later, you connect your builds
- CRDTs make sure your blocks don’t overwrite each other—they fit together logically

---

## CRDTs Merge Concurrent Edits — How?

- Every edit includes:
  - **Who did it**
  - **What they did**
  - **When they did it**
- The system builds a timeline of edits—even when out of order
- All users end up with the same final result

**Example:**

```js
A inserts "Hi" at position 0
B inserts "There" at position 0
→ CRDT merges as: "ThereHi" or "HiThere", but always the same on all devices
```

---

## Two Main Types of CRDTs

### 1. State-based (CvRDT)

- Devices send each other their **whole state**
- Merge = combine states (usually with some max or union)

🧠 Easy to reason about\
⚠️ Can be heavy on bandwidth

### 2. Operation-based (CmRDT)

- Devices send only the **changes** they made
- Requires operations to arrive in the right order

🧠 Efficient\
⚠️ Needs reliable messaging

---

### Operation-based (CmRDT) Characteristics – Matches Yjs:

```text
Replicas send operations, not full state
→ Yjs syncs inserts, deletes, and updates as delta operations encoded in binary.

Each operation has causal metadata
→ In Yjs, each change is tagged with a unique identifier (clientID:clock) and its causal dependencies.

Order of delivery matters, or must be causally recoverable
→ Yjs tracks causal order and ensures that operations are applied in a consistent, conflict-free way.

Efficient: only the deltas are shared
→ Yjs uses compact binary Uint8Array messages to sync deltas via WebSockets, WebRTC, or other transports.
```

---

## How Does Yjs Work? (Approachable)

**Yjs** is a JavaScript library that brings CRDT magic to web apps.\
It powers tools like Tiptap and ProseMirror for rich-text collaboration.

### Basic Building Block:

- A shared document called a `Y.Doc`
- Contains shared types: `Y.Text`, `Y.Array`, etc.

---

## How Yjs Works Internally (More Technical)

- Uses a CRDT called **YATA** (Yet Another Transformation Approach)
- Each operation (insert/delete) gets:
  - A **unique ID** (client + clock)
  - Causal ordering based on previous edits
- Binary encoded updates are sent between peers

---

## What Is the Message Format in Yjs?

All communication is compact binary messages.\
These are fast and efficient!

### Message Types:

```text
0 = sync
1 = update (insert/delete)
2 = awareness (cursor, name)
```

Payload is encoded with `Uint8Array`.

---

## What Data Is Stored on the Yjs Server?

By default:

- **Nothing!**
- The server only **relays** updates between users
- All data lives in the browser/app memory

Optional:

- You can plug in storage like:
  - LevelDB (Node.js)
  - IndexedDB (browser)
  - Redis (server-side)

---

## What Might We Use Instead of Yjs?

| Option                | Pros                     | Cons                            |
| --------------------- | ------------------------ | ------------------------------- |
| **Automerge**         | Simple, JSON-friendly    | Big data size, slower sync      |
| **Diamond Types**     | Fast for text editing    | Not for rich documents          |
| **Peritext**          | Semantic-rich (research) | Experimental                    |
| **OT (e.g. ShareDB)** | Battle-tested            | Complex, requires central logic |

---

## 🧬 What a Yjs CRDT Looks Like in Binary

Let’s meet the rawest version of the CRDT: the **encoded binary update** itself.

> Imagine saving your shared document as bytes. That’s what Yjs snapshots do.

### Here’s a real update:

```js
update = [
  29, 1, 234, 193, 136, 128, 15, 0, 0, 12, 6, 145, 219, 189, 208, 14, 0, 129, 207, 243, 179, 142,
  14, 23, 4, 129, 207, 243, 179, 142, 14, 0, 1, 0, 19, 129, 145, 219, 189, 208, 14, 4, 1, 0, 15,
  ...
  20, 194, 220, 132, 17, 1, 0, 19, 242, 149, 217, 22, 1, 0, 17
]
```

Don’t worry if that looks intense.

Here’s what it really means:

- It’s a compact binary encoding of **many individual Yjs operations**.
- Each block corresponds to an "Item" in the CRDT—an insert, a delete, a parent-child relation, etc.
- When you load this into a `Y.Doc`, it reconstructs the full shared state of the document, including text, attributes, positions, and structure.

Why binary?

- To reduce network size.
- To increase speed of syncs.
- To let peers exchange just the **minimum changes needed**.

This snapshot and its byte string are part of the **core of real-world CRDT collaboration**.

---

## 📂 Yjs Snapshot File: What's Inside?

Before we look at the live CRDT structure from our app, let’s look at something even more concrete: a real Yjs **snapshot file**.

We saved a snapshot of our collaborative document in a file called:

```bash
snapshot-2025-06-09T21-33-54-052Z.ysnap.json
```

This snapshot contains a compact binary representation of the **entire CRDT state** at a particular point in time. It’s structured as an encoded update array — a `Uint8Array`, to be precise.

### 🔍 What’s Actually In the File?

This file contains:

- **Encoded client and clock info** for each operation
- **Structs** representing inserts, deletes, and content
- All operations tagged with their **unique origin** and causal order
- **Parent-child structure**, even for ProseMirror XML elements

You’ll see identifiers like:

```js
id: { client: 3392198051, clock: 0 }
content: ContentType { type: [YXmlElement] }
content: ContentDeleted
```

These match exactly what we’ll see in the next section: the live CRDT data structure.

This snapshot format can be:

- **Saved to disk**
- **Loaded into another Y.Doc**
- **Synced across peers** to rebuild the exact state

Let’s now look at how this file maps directly to the internal CRDT graph.

---

## 🔍 Real CRDT Snapshot (From Our App)

We captured a live snapshot from our Yjs-powered editor. Let’s walk through the **actual CRDT structure**.

---

### 🎯 Context

In our app, we use two shared types:

```js
→ prosemirror: AbstractType   // our actual collaborative document
→ default: AbstractType       // an unused placeholder
```

Both are CRDTs managed by Yjs.

---

### 📐 The CRDT Structure

Each Yjs shared type (e.g., `prosemirror`) is made of `Items`.\
An `Item` represents an operation (insert or delete) with a unique ID and structure:

```js
Item {
  id: ID { client: 3392198051, clock: 0 },
  content: ContentType { type: [YXmlElement] },
  right: Item {
    id: ID { client: 3788306895, clock: 0 },
    content: ContentDeleted
  },
  parent: prosemirror
}
```

---

### 🔑 CRDT Features in Action

| Feature                        | Snapshot Evidence                               |
| ------------------------------ | ----------------------------------------------- |
| **Unique ID**                  | `client: 3392198051, clock: 0`                  |
| **Causal order**               | `right` and `origin` pointers                   |
| **Mergeable**                  | Every Item connects via `left`/`right`          |
| **Tombstoned deletes**         | `content: ContentDeleted`                       |
| **No central source of truth** | Sync happens across peers, not through a server |

---

### 🤯 Why This Matters

Even if clients:

- edit offline,
- come back later, or
- edit the same spot...

The structure ensures that every update merges **without conflict**.

---

### 🧼 About `default`

We also saw this in our snapshot:

```js
→ default: AbstractType
Item {
  id: ID { client: 2356320008, clock: 58 },
  content: ContentType { type: [YXmlElement] }
}
```

It exists because some Yjs APIs create unnamed shared types automatically.\
It’s unused in our case, and safe to ignore.

---

## 🧠 In Summary

This is a real CRDT:

- It’s how **Yjs implements collaboration**
- Your editor state is a **graph of causally linked Items**
- Each client maintains its own view — but merges happen without conflict

**You just saw the internal structure behind real-time collaboration.**

---

## Summary (Non-Technical + Technical Together)

CRDTs:

- Let people edit offline and sync later
- Merge changes automatically without drama
- Are the backbone of collaborative editing tools

Yjs:

- A fast, lean CRDT implementation in JavaScript
- Works well with ProseMirror/Tiptap
- Communicates via binary messages
- Stores no user data by default

---

## Resources

- [Yjs on GitHub](https://github.com/yjs/yjs)
- [Automerge](https://automerge.org)
- [Tiptap Docs](https://docs.tiptap.dev)
- [CRDT Primer](https://crdt.tech/)

---

## Thank You!

Questions?\
Let's break it down further if needed!


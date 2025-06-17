CRDTs for Collaborative Editors

By JJ Salley

---

## Welcome

This talk will cover **CRDTs**, or Conflict-Free Replicated Data Types.\
They sound technical, but we’ll walk through how they work step by step—with real examples.

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
That means:

> "A way to let multiple users change shared data at the same time without needing to check with each other."

---

## What Makes CRDTs Special?

### The Naive Way: Shared Array of Bytes

- Think: a text file shared through Dropbox
- If two people change it at once: it breaks or someone loses data

### The CRDT Way

- Each user makes changes independently
- The system tracks **intent**, **timing**, and **authorship**
- Later, it can merge everyone’s changes consistently

---

## CRDTs Merge Concurrent Edits — How?

- Every edit includes:
  - **Who did it** (client ID)
  - **What they did** (insert, delete)
  - **When they did it** (clock)
- The system uses this info to build a timeline of edits—even if received out of order

**Example:**

```js
A inserts "Hi" at position 0
B inserts "There" at position 0
→ CRDT merges as: "ThereHi" or "HiThere", but always the same everywhere
```

---

## Two Main Types of CRDTs

### 1. State-based (CvRDT)

- Devices send each other their **whole state**
- Merge = combine states (usually with some max or union)

Simple concept
More bandwidth required

### 2. Operation-based (CmRDT)

- Devices send only the **changes** they made
- Order matters or must be recoverable

Efficient
More logic required

---

## Operation-based (CmRDT) Characteristics – Matches Yjs:

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

## How Does Yjs Work? 

**Yjs** is a JavaScript library for collaborative editing.

### It uses:

- A document container (`Y.Doc`)
- Shared types: `Y.Text`, `Y.Array`, `Y.Map`, `Y.XmlFragment`
- Syncs changes across peers using binary updates

---

## How Yjs Works Internally (More Technical)

- Uses a CRDT algorithm called **YATA** (Yet Another Transformation Approach)
- Edits are structured as **Items**
  - Each has a unique `client + clock` ID
  - Causal dependencies tracked via `left`, `right`, and `origin`
  - Can be inserts, deletes, or format changes

---

## What Is the Message Format in Yjs?

### Message Types:

```text
0 = sync
1 = update (insert/delete)
2 = awareness (cursor, name)
```

Payloads are sent as compact binary chunks.

---

## What Data Is Stored on the Yjs Server?

By default:

- **Nothing**
- It relays updates between clients

You can add optional storage using:

- LevelDB (Node)
- Redis
- IndexedDB (browser).  I am doing this.

---

## What Might We Use Instead of Yjs?

| Option                | Pros                     | Cons                        |
| --------------------- | ------------------------ | --------------------------- |
| **Automerge**         | Simple, JSON-friendly    | Big data size, slower sync  |
| **Diamond Types**     | Fast for text editing    | Not for rich documents      |
| **Peritext**          | Semantic-rich (research) | Experimental                |
| **OT (e.g. ShareDB)** | Battle-tested            | Centralized logic, not CRDT |

---

## Yjs Binary Snapshot

This is the raw CRDT state encoded into a binary array:

```js
[29, 1, 234, 193, 136, 128, 15, ...]
```

What’s inside:

- Edits (as Items)
- Metadata (client IDs, clocks)
- Structure and ordering

You can **load it into Y.Doc** to rebuild the document fully.

---

## Yjs Snapshot File: What’s Inside?

A `.ysnap.json` file contains:

- A JSON object with a `Uint8Array` called `update`
- That’s the binary CRDT update

You can apply it to any new `Y.Doc` to see the full structure.

---

## Also In Practice: `verifyfilestructure.js`

We also use:

- `verifyfilestructure.js` to print the full CRDT structure
- Outputs every `Item`, its parent/child links, content, and deletion status

It shows the **graph** of Items and how Yjs stores them.

---

## Real CRDT Snapshot (From Our App)

We saved one of these snapshots from our editor:

```js
→ prosemirror: YXmlFragment
→ default: AbstractType
```

`prosemirror` holds the actual document.

---

## What Is Inside the prosemirror CRDT?

Each element is a Yjs `Item`, like this:

```js
Item {
  id: { client: 12345, clock: 0 },
  content: ContentType { type: [YXmlElement] },
  right: Item { content: ContentDeleted }
}
```

These Items connect in a chain and form a causal graph.

---

## CRDT Properties in Action

| Property          | What It Looks Like         |
| ----------------- | -------------------------- |
| Unique IDs        | `client + clock`           |
| Causal ordering   | `left`, `right` references |
| Mergeable         | Links between items        |
| Deletions         | `ContentDeleted`           |
| Distributed model | No server state            |

---

## Unused CRDT Structures

In our snapshot we also saw:

```js
→ default: AbstractType
```

It’s empty or unused — a placeholder. Safe to ignore.

---

## In Practice: Our Snapshot Viewer

We built a tool called `snapshot-viewer.js`:

- Loads a `.ysnap.json` file
- Parses the binary update
- Reconstructs the CRDT
- Prints the contents of shared types (`Y.Text`, `Y.XmlFragment`, etc.)

```bash
node snapshot-viewer.js snapshot-2025-06-09T21-33-54-052Z.ysnap.json
```

---


## Recap: CRDTs and Yjs

- CRDTs allow multiple people to **edit at the same time**, even offline
- Yjs encodes these edits as compact binary **Items** with metadata
- No server needed — everything lives on the client
- Tools like `snapshot-viewer.js` and `verifyfilestructure.js` let us inspect the real structure

---

## Resources

- [Yjs on GitHub](https://github.com/yjs/yjs)
- [Automerge](https://automerge.org)
- [Tiptap Docs](https://docs.tiptap.dev)
- [CRDT Primer](https://crdt.tech/)

---

## Thank You!

Questions? 


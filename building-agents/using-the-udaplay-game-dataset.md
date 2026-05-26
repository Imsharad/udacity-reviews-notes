# Using the UdaPlay game dataset

The UdaPlay project ships with a `games/` directory full of JSON files, one per game, each describing a title with its publisher, platforms, release dates, and a long-form description. The RAG pipeline only works if those JSONs are loaded, chunked sensibly, and embedded into the vector store. Many submissions never make it past "load the data."

## Why it matters

The semantic-search demo, the agent's internal-knowledge lookup, the citations in the final answer — all of these depend on the vector store containing real game records. If the corpus is empty or contains the wrong shape (e.g., just the titles, or just the descriptions, with no metadata), every downstream check fails together: search returns nothing, the agent falls back to web search on every query, citations are blank.

The fix is small. The JSON files have a known shape. Read them once, build text + metadata records, push them into ChromaDB.

## The pattern

```python
import json
from pathlib import Path
import chromadb

client = chromadb.PersistentClient(path="udaplay_chroma")
collection = client.get_or_create_collection(
    name="udaplay_games",
    metadata={"hnsw:space": "cosine"},
)

games_dir = Path("games")
documents, metadatas, ids = [], [], []

for game_file in sorted(games_dir.glob("*.json")):
    with open(game_file) as f:
        game = json.load(f)

    # Build a single text chunk per game. The model needs both
    # title context AND the description in one embedding.
    text = (
        f"Title: {game['title']}\n"
        f"Publisher: {game.get('publisher', 'unknown')}\n"
        f"Platforms: {', '.join(game.get('platforms', []))}\n"
        f"Released: {game.get('release_date', 'unknown')}\n\n"
        f"{game.get('description', '')}"
    )
    documents.append(text)
    metadatas.append({
        "title": game["title"],
        "publisher": game.get("publisher", "unknown"),
        "year": game.get("release_year"),
        "source_file": game_file.name,
    })
    ids.append(game_file.stem)

collection.add(documents=documents, metadatas=metadatas, ids=ids)
print(f"Loaded {len(documents)} games into the udaplay_games collection")
```

The three things this gets right that students often miss:

1. **One record per game, not per JSON field**. Splitting a single game across multiple records makes citation harder and semantic search noisier.
2. **Title + description in the same text chunk**. Embedding only the description loses the title signal; the model can't tell which game it found.
3. **Metadata captured**. The metadata fields (`title`, `publisher`, `year`, `source_file`) are how the agent later cites its sources back to the user.

## Common pitfalls

- **Dataset never loaded**: the notebook imports ChromaDB, creates a collection, but the games loop is missing or commented out. The collection ends up empty. Symptom: every semantic search returns zero results.
- **Wrong path to the games directory**: `Path("./games")` vs `Path("../games")`. Silently iterates zero files. Same symptom as above.
- **Title omitted from the embedded text**: only `game["description"]` gets embedded. The model finds "the game with the storyline about a plumber" but can't surface the title back. Citations break.
- **Each JSON field added as a separate document**: every game produces ten records (title, publisher, platforms, ...). Search returns "Nintendo" three times when the user asked for "the Mario game on N64."
- **Metadata typed as nested objects**: ChromaDB metadata values must be primitives (str, int, float, bool). Passing `metadata={"genres": ["action", "rpg"]}` raises an error or gets silently dropped depending on the client version.
- **`get_or_create_collection` vs `create_collection`**: the second raises if the collection already exists. On the second run of the notebook, the load cell errors. Use `get_or_create_collection` and accept idempotent re-loads, or delete the collection first.

## Further reading

- [ChromaDB — Add data to a collection](https://docs.trychroma.com/docs/collections/add-data) — first-party reference for the `collection.add(documents=, metadatas=, ids=)` API.
- [ChromaDB — Persistent client](https://docs.trychroma.com/docs/run-chroma/persistent-client) — how to make the collection survive a kernel restart.

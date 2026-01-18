# Tutorial: Minimalne kroki do uruchomienia projektu RAG

## Wymagania

- **Docker** i **Docker Compose** (do uruchomienia Ollama i Qdrant)
- **Node.js** (wersja 18+)
- **npm** lub **yarn**

## Krok 1: Uruchom serwisy (Docker)

```bash
docker-compose up -d
```

To uruchomi:

- **Ollama** na porcie `11434` (modele AI)
- **Qdrant** na portach `6333` i `6334` (baza wektorowa)

## Krok 2: Zainstaluj zależności Node.js

```bash
npm install
```

## Krok 3: Pobierz modele Ollama

Ponieważ Ollama działa w Dockerze, użyj `docker exec`:

```bash
# Model do embeddings (wektoryzacja tekstu)
docker exec ollama ollama pull all-minilm

# Model do chat (odpowiedzi)
docker exec ollama ollama pull llama3.2:3b
```

**Alternatywnie**, jeśli masz zainstalowany Ollama CLI lokalnie, możesz użyć:
```bash
ollama pull all-minilm
ollama pull llama3.2:3b
```

## Krok 4: Przygotuj dane

Umieść plik z wiedzą w folderze `data/`:

```bash
mkdir -p data
echo "Twoja wiedza tutaj..." > data/wiedza.txt
```

## Krok 5: Zaindeksuj dokumenty

```bash
npm run ingest
```

To:

- Wczyta plik `data/wiedza.txt`
- Podzieli go na fragmenty
- Wygeneruje embeddings
- Zapisze do Qdrant

## Krok 6: Zadaj pytanie

```bash
npm run ask "Twoje pytanie tutaj"
```

Lub bez argumentu (użyje domyślnego pytania):

```bash
npm run ask
```

## Struktura projektu

```
projekt/
├── data/              # Pliki z wiedzą (.txt)
├── src/
│   ├── ingests.ts     # Indeksowanie dokumentów
│   └── ask.ts         # Zadawanie pytań
├── docker-compose.yml # Konfiguracja Docker
└── package.json       # Zależności Node.js
```

## Minimalne pliki do stworzenia

### 1. `package.json`

```json
{
	"scripts": {
		"ingest": "ts-node src/ingests.ts",
		"ask": "ts-node src/ask.ts",
		"clear-qdrant": "ts-node src/clear-qdrant.ts"
	},
	"dependencies": {
		"@langchain/ollama": "^1.2.0",
		"@langchain/qdrant": "^1.0.1",
		"@langchain/textsplitters": "^1.0.1",
		"@langchain/core": "^1.1.13"
	},
	"devDependencies": {
		"typescript": "^5.9.3",
		"ts-node": "^10.9.2",
		"@types/node": "^25.0.8"
	}
}
```

### 2. `docker-compose.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    ports:
      - '11434:11434'
    volumes:
      - ./ollama_data:/root/.ollama

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - '6333:6333'
    volumes:
      - ./qdrant_data:/qdrant/storage
```

### 3. `tsconfig.json`

```json
{
	"compilerOptions": {
		"target": "ESNext",
		"module": "NodeNext",
		"moduleResolution": "NodeNext",
		"strict": true,
		"esModuleInterop": true
	}
}
```

### 4. `src/ingests.ts` (minimalna wersja)

```typescript
import { TextLoader } from '@langchain/classic/document_loaders/fs/text';
import { RecursiveCharacterTextSplitter } from '@langchain/textsplitters';
import { OllamaEmbeddings } from '@langchain/ollama';
import { QdrantVectorStore } from '@langchain/qdrant';

async function run() {
	const loader = new TextLoader('data/wiedza.txt');
	const docs = await loader.load();

	const splitter = new RecursiveCharacterTextSplitter({
		chunkSize: 500,
		chunkOverlap: 150,
	});
	const splits = await splitter.splitDocuments(docs);

	const embeddings = new OllamaEmbeddings({
		model: 'all-minilm',
		baseUrl: 'http://localhost:11434',
	});

	await QdrantVectorStore.fromDocuments(splits, embeddings, {
		url: 'http://localhost:6333',
		collectionName: 'my_documents',
	});

	console.log('✅ Gotowe!');
}

run();
```

### 5. `src/ask.ts` (minimalna wersja)

```typescript
import { OllamaEmbeddings, ChatOllama } from '@langchain/ollama';
import { QdrantVectorStore } from '@langchain/qdrant';
import { PromptTemplate } from '@langchain/core/prompts';
import { StringOutputParser } from '@langchain/core/output_parsers';
import { RunnableSequence } from '@langchain/core/runnables';

async function main() {
	const question = process.argv.slice(2).join(' ') || 'Twoje pytanie?';

	const embeddings = new OllamaEmbeddings({
		model: 'all-minilm',
		baseUrl: 'http://localhost:11434',
	});

	const model = new ChatOllama({
		model: 'llama3.2:3b',
		baseUrl: 'http://localhost:11434',
	});

	const vectorStore = await QdrantVectorStore.fromExistingCollection(
		embeddings,
		{
			url: 'http://localhost:6333',
			collectionName: 'my_documents',
		}
	);

	const retriever = vectorStore.asRetriever({ k: 5 });

	const template = `Odpowiedz na podstawie kontekstu:
{context}

Pytanie: {question}
Odpowiedź:`;

	const prompt = PromptTemplate.fromTemplate(template);

	const chain = RunnableSequence.from([
		{
			context: async (input: string) => {
				const docs = await retriever.invoke(input);
				return docs.map((d) => d.pageContent).join('\n\n');
			},
			question: (input: string) => input,
		},
		prompt,
		model,
		new StringOutputParser(),
	]);

	const response = await chain.invoke(question);
	console.log(response);
}

main();
```

## Szybki start (wszystkie kroki)

```bash
# 1. Uruchom Docker
docker-compose up -d

# 2. Zainstaluj zależności
npm install

# 3. Pobierz modele (przez Docker)
docker exec ollama ollama pull all-minilm
docker exec ollama ollama pull llama3.2:3b

# 4. Przygotuj dane
mkdir -p data
echo "Twoja wiedza..." > data/wiedza.txt

# 5. Zaindeksuj
npm run ingest

# 6. Zadaj pytanie
npm run ask "Twoje pytanie?"
```

## Rozwiązywanie problemów

- **Ollama nie odpowiada**: Sprawdź czy Docker działa: `docker ps`
- **Model nie znaleziony**: Uruchom `docker exec ollama ollama pull nazwa_modelu`
- **Qdrant nie działa**: Sprawdź logi: `docker logs qdrant`
- **Błąd TypeScript**: Upewnij się że masz `ts-node` zainstalowane

## Co dalej?

- Zmień modele w `ingests.ts` i `ask.ts` (np. na większe modele)
- Dodaj więcej plików do indeksowania
- Dostosuj `chunkSize` i `chunkOverlap` w `ingests.ts`
- Zmodyfikuj prompt w `ask.ts` dla lepszych odpowiedzi

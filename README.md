# Tutorial: Minimal Steps to Run RAG Project

## Requirements

- **Docker** and **Docker Compose** (to run Ollama and Qdrant)
- **Node.js** (version 18+)
- **npm** or **yarn**

## Step 1: Start Services (Docker)

```bash
docker-compose up -d
```

This will start:

- **Ollama** on port `11434` (AI models)
- **Qdrant** on ports `6333` and `6334` (vector database)

## Step 2: Install Node.js Dependencies

```bash
npm install
```

## Step 3: Download Ollama Models

Since Ollama runs in Docker, use `docker exec`:

```bash
# Model for embeddings (text vectorization)
docker exec ollama ollama pull all-minilm

# Model for chat (responses)
docker exec ollama ollama pull llama3.2:3b
```

**Alternatively**, if you have Ollama CLI installed locally, you can use:
```bash
ollama pull all-minilm
ollama pull llama3.2:3b
```

## Step 4: Prepare Data

Place your knowledge file in the `data/` folder:

```bash
mkdir -p data
echo "Your knowledge here..." > data/wiedza.txt
```

## Step 5: Index Documents

```bash
npm run ingest
```

This will:

- Load the file `data/wiedza.txt`
- Split it into chunks
- Generate embeddings
- Save to Qdrant

## Step 6: Ask a Question

```bash
npm run ask "Your question here"
```

Or without arguments (will use default question):

```bash
npm run ask
```

## Project Structure

```
project/
├── data/              # Knowledge files (.txt)
├── src/
│   ├── ingests.ts     # Document indexing
│   └── ask.ts         # Asking questions
├── docker-compose.yml # Docker configuration
└── package.json       # Node.js dependencies
```

## Minimal Files to Create

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

### 4. `src/ingests.ts` (minimal version)

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

	console.log('✅ Done!');
}

run();
```

### 5. `src/ask.ts` (minimal version)

```typescript
import { OllamaEmbeddings, ChatOllama } from '@langchain/ollama';
import { QdrantVectorStore } from '@langchain/qdrant';
import { PromptTemplate } from '@langchain/core/prompts';
import { StringOutputParser } from '@langchain/core/output_parsers';
import { RunnableSequence } from '@langchain/core/runnables';

async function main() {
	const question = process.argv.slice(2).join(' ') || 'Your question?';

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

	const template = `Answer based on context:
{context}

Question: {question}
Answer:`;

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

## Quick Start (All Steps)

```bash
# 1. Start Docker
docker-compose up -d

# 2. Install dependencies
npm install

# 3. Download models (via Docker)
docker exec ollama ollama pull all-minilm
docker exec ollama ollama pull llama3.2:3b

# 4. Prepare data
mkdir -p data
echo "Your knowledge..." > data/wiedza.txt

# 5. Index
npm run ingest

# 6. Ask a question
npm run ask "Your question?"
```

## Troubleshooting

- **Ollama not responding**: Check if Docker is running: `docker ps`
- **Model not found**: Run `docker exec ollama ollama pull model_name`
- **Qdrant not working**: Check logs: `docker logs qdrant`
- **TypeScript error**: Make sure you have `ts-node` installed

## What's Next?

- Change models in `ingests.ts` and `ask.ts` (e.g., to larger models)
- Add more files for indexing
- Adjust `chunkSize` and `chunkOverlap` in `ingests.ts`
- Modify the prompt in `ask.ts` for better answers

# Smart Story Agent

A Next.js application that transforms news articles into structured, digestible content using Claude AI. Enter any article URL and get an AI-generated summary, key highlights, organized fact sections, and engagement scoring.

## Tech Stack

### Framework & Runtime
- **Next.js 15.2.4** with App Router (React Server Components)
- **React 19.0.0**
- **TypeScript 5** with strict mode

### AI & Content Processing
- **Anthropic SDK (v0.39.0)** - Claude API integration for article analysis
- **Mozilla Readability (v0.6.0)** - Article content extraction from web pages
- **JSDOM (v26.0.0)** - Server-side DOM parsing for HTML processing
- **dice-coefficient (v2.1.1)** - Text similarity scoring between original and extracted content

### UI & Styling
- **Tailwind CSS 4** - Utility-first styling
- **Framer Motion 12.6.3** - Animations and transitions
- **Geist Font** - Typography via Google Fonts

### Analytics
- **PostHog** - Client (v1.236.2) and server-side (v4.11.7) analytics

## Architecture

### Agent Implementation

The AI agent lives in a single API route at `src/app/api/process-article/route.ts`. This endpoint handles the complete article processing pipeline:

**Processing Flow:**
1. **URL Validation** - Validates the incoming article URL format
2. **Content Fetching** - Fetches the article HTML with a 15-second timeout using a custom user agent (`SmartStorySuiteBot/1.0`)
3. **Content Extraction** - Uses Mozilla Readability to extract the main article content, with a fallback to raw body text if extraction fails
4. **Metadata Scraping** - Extracts Open Graph images, publication dates, and author information from meta tags
5. **AI Analysis** - Sends the extracted content to Claude for structured analysis
6. **Similarity Scoring** - Calculates a Dice coefficient between the original text and fact sections to measure content preservation

**Model Configuration:**
- Model: `claude-3-haiku-20240307`
- Max Tokens: 4000
- Temperature: 0.1 (low variance for consistent JSON output)

### Output Schema

The agent returns structured JSON with the following format:

```typescript
interface StoryData {
  title: string;
  source: string;
  date: string;
  summary: string;              // 2-4 sentence overview
  highlights: string[];         // Exactly 3 highlights, max 10 words each
  factSections: Array<{
    id: string;                 // Auto-generated slug from title
    title: string;              // Dynamic section title based on content
    content: string;            // ~150 word chunks of original text
  }>;
  spiceScore: {
    s: number;                  // Scannability (1-5)
    p: number;                  // Personalization (1-5)
    i: number;                  // Interactivity (1-5)
    c: number;                  // Curation (1-5)
    e: number;                  // Emotion (1-5)
    total: number;              // Sum (5-25)
    justifications: object;
  } | null;
  similarityScore: number;      // Dice coefficient (0-1)
  primaryImage: string | null;
  additionalImages: string[];
  author: string | null;
}
```

### Data Sources

The application processes articles on-demand without persistent storage or indexing:

- **Input**: Any publicly accessible HTTP/HTTPS article URL
- **Extraction**: Mozilla Readability parses and extracts main content, falling back to raw HTML body if needed (minimum 150 characters required)
- **Images**: Primary image from `og:image` meta tag; additional images extracted from article content (up to 10, validated for size and protocol)
- **No Vector Database**: Text similarity is computed using the Dice coefficient algorithm (bigram-based), not semantic embeddings
- **No Crawling/Indexing**: Purely stateless request-response processing

## Project Structure

```
src/
├── app/
│   ├── api/
│   │   └── process-article/
│   │       └── route.ts       # AI agent API endpoint
│   ├── layout.tsx             # Root layout with PostHog provider
│   ├── page.tsx               # Main UI (SmartStorySuite component)
│   └── globals.css            # Tailwind config and theme variables
├── components/
│   └── PostHogProvider.tsx    # Analytics context wrapper
└── lib/
    └── posthog.ts             # PostHog client initialization
```

## UI Components

The main interface (`src/app/page.tsx`) renders a three-column layout:

### Left Sidebar - Section Navigation
- Clickable buttons for each fact section
- Active state highlighting
- Smooth scroll to selected content

### Center Content - Story Display
- **Highlights Component**: 3 bullet-point key takeaways
- **Summary Component**: 2-4 sentence article overview
- **Fact Sections**: Dynamically titled content blocks (~150 words each)
- Toggle between Summary View (single section) and Detailed View (all sections)

### Right Sidebar - Metadata & Scores
- **Story Details**: Published date, source, author
- **SPICE Score Display**: Engagement metrics (Scannability, Personalization, Interactivity, Curation, Emotion)
- **Similarity Score**: Content preservation percentage with color coding (green ≥80%, yellow 60-79%, red <60%)
- **Image Gallery**: Primary image with modal enlargement, additional images grid

### Additional Features
- Dark/light mode toggle with system preference detection
- Framer Motion animations throughout
- Responsive design for mobile and desktop
- Error handling with user-friendly alerts

## Environment Variables

Create a `.env.local` file with:

```bash
# Required - Claude API key for article processing
ANTHROPIC_API_KEY=your_anthropic_api_key

# Required - PostHog analytics
NEXT_PUBLIC_POSTHOG_KEY=your_posthog_key
```

## Getting Started

1. **Install dependencies:**
```bash
npm install
```

2. **Set up environment variables:**
```bash
cp .env.example .env.local
# Edit .env.local with your API keys
```

3. **Run the development server:**
```bash
npm run dev
```

4. **Open [http://localhost:3000](http://localhost:3000)**

## Scripts

```bash
npm run dev      # Start development server
npm run build    # Create production build
npm start        # Run production server
npm run lint     # Run ESLint
```

## Deployment

Designed for deployment on **Vercel**:

1. Push your code to a Git repository
2. Import the project in Vercel
3. Add environment variables (`ANTHROPIC_API_KEY`, `NEXT_PUBLIC_POSTHOG_KEY`)
4. Deploy

The `next.config.ts` includes URL rewrites for PostHog API endpoints to avoid ad-blocker interference.

## How It Works

1. User enters an article URL in the input field
2. The frontend sends a POST request to `/api/process-article`
3. The API fetches the article and extracts content using Readability
4. Article text is sent to Claude with a structured prompt requesting JSON output
5. Claude analyzes the content and returns:
   - A concise summary and 3 key highlights
   - Fact sections with meaningful titles (preserving original text)
   - SPICE engagement scores with justifications
6. The API calculates a similarity score and returns the complete response
7. The frontend renders the structured data with animations

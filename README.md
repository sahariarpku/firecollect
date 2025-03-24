# FireCollect Magic - Research Paper Analysis & Chat Platform

FireCollect Magic is an advanced research paper analysis platform that helps researchers and academics efficiently manage, analyze, and interact with academic papers and PDF documents using AI-powered features.

## Features

### 1. Research Paper Search & Analysis
- Search and analyze academic papers across multiple sources
- View detailed paper summaries and metadata
- Track search history for easy reference
- Export research data to Excel format

### 2. PDF Management
- Upload and analyze individual PDFs or batches
- Automatic extraction of key information:
  - Title and authors
  - Abstract and background
  - Research questions
  - Major findings
  - Suggestions and conclusions
- Organize PDFs into batches for better management

### 3. AI-Powered Chat Interface
- Interactive chat with AI about your research papers
- Context-aware responses based on paper content
- Support for:
  - Individual paper analysis
  - Batch PDF discussions
  - Research query exploration
- Clean, user-friendly interface with:
  - Markdown support
  - Code block formatting
  - Copy-to-clipboard functionality
  - Message history

## Getting Started

### Prerequisites
- Node.js (v16 or higher)
- npm or yarn package manager
- Supabase account for database
- AI service API key (OpenAI, Google, Anthropic, etc.)

### Installation

1. Clone the repository:
```bash
git clone https://github.com/sahariarpku/firecollect.git
cd firecollect-magic
```

2. Install dependencies:
```bash
npm install
# or
yarn install
```

3. Set up environment variables:
find file /firecollect/src/integrations/supabase/client.ts:
```client.ts:
const SUPABASE_URL = "your url here";
const SUPABASE_PUBLISHABLE_KEY = "your key here";

```
find /firecollect/supabase/config.toml and add project id:
```config.toml
project_id = ""
```
4. Set up your database schema. You'll need to run these SQL commands in your Supabase SQL editor:
```sql:
-- Enable necessary extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create storage bucket for PDFs
INSERT INTO storage.buckets (id, name, public)
VALUES ('pdfs', 'pdfs', false)
ON CONFLICT (id) DO NOTHING;

-- Create RLS (Row Level Security) policy for storage
CREATE POLICY "Public Access" ON storage.objects
  FOR SELECT USING (bucket_id = 'pdfs');

-- Create tables with proper relationships and indexes

-- Create searches table
CREATE TABLE IF NOT EXISTS searches (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    query TEXT NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create papers table
CREATE TABLE IF NOT EXISTS papers (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL,
    author TEXT NOT NULL,
    year INTEGER,
    abstract TEXT,
    doi TEXT,
    search_id UUID REFERENCES searches(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create index for search_id in papers
CREATE INDEX IF NOT EXISTS idx_papers_search_id ON papers(search_id);

-- Create pdf_uploads table
CREATE TABLE IF NOT EXISTS pdf_uploads (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    filename TEXT NOT NULL,
    title TEXT,
    authors TEXT,
    year INTEGER,
    doi TEXT,
    background TEXT,
    full_text TEXT,
    markdown_content TEXT,
    research_question TEXT,
    major_findings TEXT,
    suggestions TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create pdf_batches table
CREATE TABLE IF NOT EXISTS pdf_batches (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create batch_pdfs table (junction table for many-to-many relationship)
CREATE TABLE IF NOT EXISTS batch_pdfs (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    batch_id UUID REFERENCES pdf_batches(id) ON DELETE CASCADE,
    pdf_id UUID REFERENCES pdf_uploads(id) ON DELETE CASCADE,
    UNIQUE(batch_id, pdf_id)
);

-- Create indexes for batch_pdfs
CREATE INDEX IF NOT EXISTS idx_batch_pdfs_batch_id ON batch_pdfs(batch_id);
CREATE INDEX IF NOT EXISTS idx_batch_pdfs_pdf_id ON batch_pdfs(pdf_id);

-- Create firecrawl_api_keys table
CREATE TABLE IF NOT EXISTS firecrawl_api_keys (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    api_key TEXT NOT NULL,
    user_id UUID,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create ai_models table
CREATE TABLE IF NOT EXISTS ai_models (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name TEXT NOT NULL,
    provider TEXT NOT NULL,
    api_key TEXT NOT NULL,
    base_url TEXT,
    model_name TEXT NOT NULL,
    is_default BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create RLS policies for all tables

-- Searches table policies
ALTER TABLE searches ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for searches" ON searches
    FOR ALL USING (true);

-- Papers table policies
ALTER TABLE papers ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for papers" ON papers
    FOR ALL USING (true);

-- PDF uploads table policies
ALTER TABLE pdf_uploads ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for pdf_uploads" ON pdf_uploads
    FOR ALL USING (true);

-- PDF batches table policies
ALTER TABLE pdf_batches ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for pdf_batches" ON pdf_batches
    FOR ALL USING (true);

-- Batch PDFs table policies
ALTER TABLE batch_pdfs ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for batch_pdfs" ON batch_pdfs
    FOR ALL USING (true);

-- Firecrawl API keys table policies
ALTER TABLE firecrawl_api_keys ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for firecrawl_api_keys" ON firecrawl_api_keys
    FOR ALL USING (true);

-- AI models table policies
ALTER TABLE ai_models ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Enable all access for ai_models" ON ai_models
    FOR ALL USING (true);

-- Create functions for handling timestamps
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Create triggers for updating timestamps
CREATE TRIGGER update_ai_models_updated_at
    BEFORE UPDATE ON ai_models
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_firecrawl_api_keys_updated_at
    BEFORE UPDATE ON firecrawl_api_keys
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Create function to ensure only one default AI model
CREATE OR REPLACE FUNCTION ensure_single_default_ai_model()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.is_default THEN
        UPDATE ai_models SET is_default = false WHERE id != NEW.id;
    END IF;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Create trigger for default AI model
CREATE TRIGGER ensure_single_default_ai_model_trigger
    BEFORE INSERT OR UPDATE ON ai_models
    FOR EACH ROW
    EXECUTE FUNCTION ensure_single_default_ai_model();

-- Create indexes for frequently accessed columns
CREATE INDEX IF NOT EXISTS idx_pdf_uploads_created_at ON pdf_uploads(created_at);
CREATE INDEX IF NOT EXISTS idx_pdf_batches_timestamp ON pdf_batches(timestamp);
CREATE INDEX IF NOT EXISTS idx_searches_timestamp ON searches(timestamp);

-- Grant necessary permissions
GRANT ALL ON ALL TABLES IN SCHEMA public TO postgres, anon, authenticated, service_role;
```
4. Start the development server:
```bash
npm i
npm run dev
# or
yarn dev
```

## Usage Guide

### Research Paper Search
1. Enter your research query in the main search bar
2. View results in the organized list
3. Click on papers to view details
4. Use the chat icon to discuss specific papers with AI
5. Export results using the download button

### PDF Management
1. Upload PDFs:
   - Single PDF upload
   - Batch upload for multiple files
2. View PDF analysis in the dashboard
3. Organize PDFs into batches
4. Use AI chat to discuss PDF contents

### Using the AI Chat
1. Click the chat icon on any research item
2. Type your question in the input field
3. View AI responses with proper formatting
4. Copy responses using the copy button
5. Browse chat history in the conversation

### Search History
- Access your search history in the sidebar
- View past searches, PDFs, and batches
- Rerun previous searches
- Delete unwanted history items

## Technical Stack

- **Frontend**:
  - React with TypeScript
  - Vite for build tooling
  - Tailwind CSS for styling
  - shadcn/ui for UI components
  - Framer Motion for animations

- **Backend**:
  - Supabase for database and authentication
  - AI service integration (OpenAI/Google/Anthropic)

- **Key Libraries**:
  - `@mendable/firecrawl-js` for paper analysis
  - Various AI service SDKs
  - Excel export utilities

## Contributing

We welcome contributions! Please feel free to submit pull requests, create issues, or suggest improvements.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For support, please:
1. Check the documentation
2. Create an issue in the repository
3. Contact the development team

---

Built with ❤️ using React, TypeScript, and AI

# CipherVault — Zero-Knowledge Password Manager

A fully offline-capable, zero-knowledge encrypted password manager built as a Progressive Web App.

## Security Architecture

- **Encryption**: AES-256-GCM via Web Crypto API (no third-party crypto libraries)
- **Key Derivation**: PBKDF2 with SHA-256, 600,000 iterations
- **Zero Knowledge**: Server stores only encrypted ciphertext — your master password never leaves your device
- **Auto-Lock**: Vault locks after 5 minutes of inactivity or when the tab is hidden/closed
- **Fresh IVs**: Every save generates a new random 12-byte Initialization Vector
- **Memory Safety**: Sensitive buffers are zeroized after use; CryptoKey is non-extractable

## Quick Start

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build
```

## Phase 2: Enable Cloud Sync (Optional)

1. Create a free project at [supabase.com](https://supabase.com)
2. Copy `.env.example` to `.env` and fill in your Supabase URL and anon key
3. Run the following SQL in your Supabase SQL Editor:

```sql
-- Create encrypted vault table
CREATE TABLE IF NOT EXISTS public.vault_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE DEFAULT auth.uid(),
    encrypted_data TEXT NOT NULL,
    iv TEXT NOT NULL,
    version INT NOT NULL DEFAULT 1,
    deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT timezone('utc'::text, now()),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT timezone('utc'::text, now())
);

-- Indexes for performance
CREATE INDEX IF NOT EXISTS idx_vault_items_user_id ON public.vault_items(user_id);
CREATE INDEX IF NOT EXISTS idx_vault_items_user_updated ON public.vault_items(user_id, updated_at DESC);

-- Auto-update timestamp trigger
CREATE OR REPLACE FUNCTION public.handle_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = timezone('utc'::text, now());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER set_vault_items_updated_at
    BEFORE UPDATE ON public.vault_items
    FOR EACH ROW
    EXECUTE FUNCTION public.handle_updated_at();

-- Row Level Security (users can only access their own data)
ALTER TABLE public.vault_items ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own vault items" ON public.vault_items
    FOR SELECT TO authenticated
    USING (user_id = (SELECT auth.uid()));

CREATE POLICY "Users can insert own vault items" ON public.vault_items
    FOR INSERT TO authenticated
    WITH CHECK (user_id = (SELECT auth.uid()));

CREATE POLICY "Users can update own vault items" ON public.vault_items
    FOR UPDATE TO authenticated
    USING (user_id = (SELECT auth.uid()))
    WITH CHECK (user_id = (SELECT auth.uid()));

CREATE POLICY "Users can delete own vault items" ON public.vault_items
    FOR DELETE TO authenticated
    USING (user_id = (SELECT auth.uid()));

-- Enable Realtime for cross-device sync
ALTER TABLE public.vault_items REPLICA IDENTITY FULL;
ALTER PUBLICATION supabase_realtime ADD TABLE public.vault_items;
```

## PWA Installation

- **Desktop**: Click the install icon in Chrome/Edge address bar
- **Android**: Tap "Add to Home Screen" in Chrome menu
- **iOS**: Tap Share → "Add to Home Screen" in Safari

## Data Safety

- **Encrypted Export**: Download your entire vault as an encrypted `.ciphervault` backup file
- **Encrypted Import**: Restore from any backup file
- **Offline-First**: All data stored locally in IndexedDB — works without internet
- **Cloud Sync**: Optional Supabase sync — server sees only encrypted noise

## Tech Stack

| Layer | Technology |
|:---|:---|
| Framework | React 19 + TypeScript |
| Styling | Tailwind CSS v4 |
| Build | Vite |
| Crypto | Web Crypto API (native) |
| Local Storage | IndexedDB via idb |
| Cloud Backend | Supabase (optional) |

# DUSA Tablet - Database Schema Reference

> **PURPOSE**: Complete database schema for AI to understand data structures and write adapter queries.

---

## Core Tablet Tables

### tablet_migrations

Tracks executed migrations.

```sql
CREATE TABLE tablet_migrations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    migration VARCHAR(255) NOT NULL UNIQUE,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

---

## Mail System Tables

### tablet_mail_users

Email account registration.

```sql
CREATE TABLE tablet_mail_users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    identifier VARCHAR(64) NOT NULL,          -- Player identifier (citizenid/identifier)
    email VARCHAR(100) NOT NULL UNIQUE,       -- Email address (john.doe@tablet.mail)
    display_name VARCHAR(100),                -- Display name
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_identifier (identifier),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### tablet_mail_threads

Email conversation threads.

```sql
CREATE TABLE tablet_mail_threads (
    id INT AUTO_INCREMENT PRIMARY KEY,
    subject VARCHAR(255) NOT NULL,            -- Thread subject
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_updated (updated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### tablet_mail_thread_participants

Links users to threads (many-to-many).

```sql
CREATE TABLE tablet_mail_thread_participants (
    id INT AUTO_INCREMENT PRIMARY KEY,
    thread_id INT NOT NULL,
    user_id INT NOT NULL,
    is_read TINYINT(1) DEFAULT 0,             -- Has user read latest?
    archived TINYINT(1) DEFAULT 0,            -- Is archived for this user?
    last_read_at TIMESTAMP NULL,

    FOREIGN KEY (thread_id) REFERENCES tablet_mail_threads(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES tablet_mail_users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_participant (thread_id, user_id),
    INDEX idx_user_archived (user_id, archived)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### tablet_mail_messages

Individual email messages within threads.

```sql
CREATE TABLE tablet_mail_messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    thread_id INT NOT NULL,
    sender_id INT NOT NULL,
    body TEXT NOT NULL,                       -- Message content
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (thread_id) REFERENCES tablet_mail_threads(id) ON DELETE CASCADE,
    FOREIGN KEY (sender_id) REFERENCES tablet_mail_users(id) ON DELETE CASCADE,
    INDEX idx_thread_created (thread_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

---

## News System Tables

### tablet_news_channels

News categories/channels.

```sql
CREATE TABLE tablet_news_channels (
    id VARCHAR(50) PRIMARY KEY,               -- Channel slug (e.g., 'lspd', 'weazel')
    name VARCHAR(100) NOT NULL,               -- Display name
    description TEXT,
    icon VARCHAR(50),                         -- SF Symbol
    color VARCHAR(7),                         -- Hex color (#RRGGBB)
    job VARCHAR(50),                          -- Job that can write
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### tablet_news_articles

News articles.

```sql
CREATE TABLE tablet_news_articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    channel_id VARCHAR(50) NOT NULL,
    author_identifier VARCHAR(64) NOT NULL,   -- Writer's identifier
    author_name VARCHAR(100),                 -- Cached author name
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    image_url TEXT,                           -- Featured image
    media JSON,                               -- Additional media array
    published TINYINT(1) DEFAULT 1,           -- Is published?
    views INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (channel_id) REFERENCES tablet_news_channels(id) ON DELETE CASCADE,
    INDEX idx_channel_published (channel_id, published, created_at),
    INDEX idx_author (author_identifier)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

---

## Telemetry Tables

### dusatrace_migrations

Telemetry migration tracking.

```sql
CREATE TABLE dusatrace_migrations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    migration VARCHAR(255) NOT NULL UNIQUE,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### dusatrace_traces

Historical trace storage.

```sql
CREATE TABLE dusatrace_traces (
    id INT AUTO_INCREMENT PRIMARY KEY,
    trace_id VARCHAR(50) NOT NULL UNIQUE,     -- Unique trace ID
    action VARCHAR(100) NOT NULL,             -- Operation type
    player_id INT,                            -- Source player
    player_identifier VARCHAR(64),
    final_status ENUM('SUCCESS', 'ERROR', 'TIMEOUT') NOT NULL,
    duration_ms INT,                          -- Total duration
    step_count INT,                           -- Number of steps
    steps JSON,                               -- Step details array
    initial_data JSON,
    final_data JSON,
    resource VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_action (action),
    INDEX idx_status (final_status),
    INDEX idx_created (created_at),
    INDEX idx_player (player_identifier)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

---

## Phone System Tables (External - Reference Only)

These are from phone resources, NOT created by dusa_tablet. Adapters query these.

### lb-phone Tables

```sql
-- Users
phone_phones (id, phone_number, owner)

-- Contacts
phone_contacts (id, owner, number, name, avatar)

-- Messages (grouped by channel)
phone_message_channels (id, participants JSON)
phone_messages (id, channel_id, sender, content, timestamp, is_read)
```

### qb-phone Tables

```sql
-- Messages stored in player table as JSON
players.phone_data JSON -> {
    messages: {
        [number]: [
            { sender, content, time, read }
        ]
    },
    contacts: [
        { name, number, iban }
    ]
}
```

### qs-smartphone Tables

```sql
-- Users
smartphone_users (id, identifier, phone_number)

-- Contacts
smartphone_contacts (id, identifier, contact_number, contact_name)

-- Messages
smartphone_messages (id, sender, receiver, message, timestamp, read)
```

---

## Common Queries for Adapters

### Get Player's Phone Number

```lua
-- lb-phone
SELECT phone_number FROM phone_phones WHERE owner = ?

-- qs-smartphone
SELECT phone_number FROM smartphone_users WHERE identifier = ?
```

### Get Conversations

```lua
-- lb-phone (channel-based)
SELECT
    mc.id,
    mc.participants,
    (SELECT content FROM phone_messages WHERE channel_id = mc.id ORDER BY timestamp DESC LIMIT 1) as last_message,
    (SELECT timestamp FROM phone_messages WHERE channel_id = mc.id ORDER BY timestamp DESC LIMIT 1) as last_time,
    (SELECT COUNT(*) FROM phone_messages WHERE channel_id = mc.id AND sender != ? AND is_read = 0) as unread
FROM phone_message_channels mc
WHERE JSON_CONTAINS(mc.participants, ?)
ORDER BY last_time DESC

-- qs-smartphone (direct messages)
SELECT DISTINCT
    CASE WHEN sender = ? THEN receiver ELSE sender END as other_number,
    (SELECT message FROM smartphone_messages sm2
     WHERE (sm2.sender = ? AND sm2.receiver = other_number)
        OR (sm2.receiver = ? AND sm2.sender = other_number)
     ORDER BY timestamp DESC LIMIT 1) as last_message
FROM smartphone_messages
WHERE sender = ? OR receiver = ?
```

### Get Messages in Conversation

```lua
-- lb-phone
SELECT * FROM phone_messages
WHERE channel_id = ?
ORDER BY timestamp ASC

-- qs-smartphone
SELECT * FROM smartphone_messages
WHERE (sender = ? AND receiver = ?) OR (sender = ? AND receiver = ?)
ORDER BY timestamp ASC
```

---

## Data Format Requirements

### Conversation Object (for NUI)

```lua
{
    id = "conv_123",              -- Unique identifier
    phone_number = "555-1234",    -- Other party's number
    display_name = "John Doe",    -- Contact name or number
    last_message = "Hey there",   -- Preview text
    last_message_time = 1705320000, -- Unix timestamp
    unread_count = 3,             -- Badge count
    is_muted = false,             -- Mute status
}
```

### Message Object (for NUI)

```lua
{
    id = "msg_456",               -- Unique identifier
    content = "Hello!",           -- Message text
    timestamp = 1705320000,       -- Unix timestamp
    is_sender = true,             -- Did current user send this?
    is_read = true,               -- Read status
}
```

### Contact Object (for NUI)

```lua
{
    id = "contact_789",           -- Unique identifier
    name = "John Doe",            -- Display name
    phone_number = "555-1234",    -- Phone number
    avatar = nil,                 -- Avatar URL (optional)
}
```

---

## Migration Best Practices

### Creating New Migration

1. Create file: `sql/migrations/NNN_description.sql`
2. Use incremental number (e.g., 004, 005)
3. Include IF NOT EXISTS checks
4. Use utf8mb4_general_ci collation

**Example**:
```sql
-- sql/migrations/004_add_notes_table.sql

CREATE TABLE IF NOT EXISTS tablet_notes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    identifier VARCHAR(64) NOT NULL,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_identifier (identifier)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### Altering Existing Table

```sql
-- sql/migrations/005_add_notes_color.sql

ALTER TABLE tablet_notes
ADD COLUMN color VARCHAR(7) DEFAULT '#FFFFFF' AFTER content;
```

**NEVER** modify existing migration files - always create new ones.

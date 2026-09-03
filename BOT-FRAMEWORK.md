Telegram Bot Setup Framework (No AI) — TypeScript

This document defines a robust, modular framework for building deterministic Telegram bots with TypeScript. It establishes a clear path from incoming Telegram updates to routing, business logic, persistence, and user-facing responses.

The framework intentionally excludes AI integration. Its purpose is to provide predictable workflows, explicit state management, reliable API interactions, and maintainable application boundaries.

1. Architecture Overview

The framework follows a Layered Architecture to keep responsibilities explicit and independent.

Gateway Layer

Receives incoming updates through webhooks or long polling from the Telegram Bot API.

Router Layer

Determines which handler should process each update based on commands, callback data, inline queries, text content, or other Telegram update types.

Service Layer

Owns business rules, validation, workflow execution, and external API interactions. Handlers should delegate application logic to this layer rather than implementing it directly.

Data Layer

Provides persistence for users, workflow state, configuration, and application logs.

Core Components

Bot Core: Initializes the bot with its token and establishes the Telegram connection.

Context Manager: Provides Telegram update information together with application metadata such as the current user, chat, workflow state, and database access.

Handler Registry: Provides a central place to register command, message, callback, and other update handlers.

The intended request path is:

Telegram
   │
   ▼
Gateway
   │
   ▼
Router
   │
   ▼
Handler
   │
   ▼
Service
   │
   ▼
Database / External API
   │
   ▼
Response

This separation keeps Telegram-specific concerns at the edge of the application while keeping business logic reusable and testable.

---

2. Technology Stack

- Language: TypeScript 5+
- Runtime: Node.js 20+
- Telegram Library: grammY "v1.45.1"
- Database: SQLite for lightweight deployments or PostgreSQL for production
- ORM: Prisma
- Async Framework: Native JavaScript/TypeScript "async"/"await"
- Environment Variables: dotenv
- Testing: Vitest

---

3. Project Structure

telegram_bot_framework/
├── src/
│   ├── index.ts                 # Entry point
│   ├── config.ts                # Environment variables and settings
│   │
│   ├── bot/
│   │   ├── index.ts             # Bot initialization
│   │   └── middleware.ts        # Optional middleware
│   │
│   ├── handlers/
│   │   ├── index.ts
│   │   ├── start.ts             # /start command
│   │   ├── commands.ts          # Other commands
│   │   └── callbacks.ts         # Inline keyboard callbacks
│   │
│   ├── services/
│   │   ├── userService.ts       # User management logic
│   │   └── taskService.ts       # Business logic
│   │
│   ├── models/
│   │   └── user.ts              # Application-level user model
│   │
│   ├── utils/
│   │   ├── logger.ts            # Logging configuration
│   │   ├── helpers.ts           # Utility functions
│   │   └── stateManager.ts      # State management
│   │
│   └── types/
│       └── index.ts             # Shared TypeScript types
│
├── prisma/
│   └── schema.prisma            # Database schema
│
├── tests/
│   └── handlers.test.ts
│
├── package.json
├── tsconfig.json
├── .env
├── .gitignore
├── Dockerfile
└── README.md

---

4. Implementation Details

4.1 Configuration

Use environment variables to manage secrets and settings.

"src/config.ts"

import "dotenv/config";

export const Config = {
  BOT_TOKEN: process.env.BOT_TOKEN,

  DATABASE_URL:
    process.env.DATABASE_URL ?? "file:./bot.db",

  LOG_LEVEL:
    process.env.LOG_LEVEL ?? "info",
} as const;

if (!Config.BOT_TOKEN) {
  throw new Error("BOT_TOKEN environment variable is required.");
}

---

4.2 Core Bot Initialization

"src/bot/index.ts"

import { Bot, Context } from "grammy";
import { Config } from "../config";
import { registerStartHandler } from "../handlers/start";
import { registerCommandHandlers } from "../handlers/commands";
import { registerCallbackHandlers } from "../handlers/callbacks";

export function buildBot(): Bot {
  const bot = new Bot(Config.BOT_TOKEN);

  // Register command handlers
  registerStartHandler(bot);
  registerCommandHandlers(bot);

  // Register callback handlers
  registerCallbackHandlers(bot);

  // Global error handler
  bot.catch(async (error) => {
    console.error("Telegram bot error:", error);

    const ctx = error.ctx;

    try {
      await ctx.reply(
        "An error occurred. Please try again."
      );
    } catch (replyError) {
      console.error(
        "Failed to send error message:",
        replyError
      );
    }
  });

  return bot;
}

"src/index.ts"

import { buildBot } from "./bot";

async function main(): Promise<void> {
  const bot = buildBot();

  console.log("Starting Telegram bot...");

  await bot.start();
}

main().catch((error) => {
  console.error("Failed to start bot:", error);
  process.exit(1);
});

---

4.3 Handler Implementation

Start Command

"src/handlers/start.ts"

import { Bot, Context } from "grammy";

export function registerStartHandler(
  bot: Bot<Context>
): void {
  bot.command("start", async (ctx) => {
    const user = ctx.from;
    const chat = ctx.chat;

    if (!user || !chat) {
      return;
    }

    // Save user to database if new.
    // Example:
    // await userService.getOrCreateUser(...);

    await ctx.api.sendMessage(
      chat.id,
      `Welcome, ${user.first_name}! This is a non-AI Telegram bot framework.`
    );
  });
}

---

4.4 Command Handlers

"src/handlers/commands.ts"

import { Bot, Context } from "grammy";

export function registerCommandHandlers(
  bot: Bot<Context>
): void {
  bot.command("help", async (ctx) => {
    await ctx.reply(
      "Available commands:\n\n" +
      "/start - Start the bot\n" +
      "/help - Show this help message"
    );
  });

  // Fallback for ordinary text messages
  bot.on("message:text", async (ctx) => {
    const text = ctx.message.text;

    await ctx.reply(
      `You sent: ${text}`
    );
  });
}

---

4.5 Callback Handler

Inline keyboards can be used to provide interactive buttons.

"src/handlers/callbacks.ts"

import {
  Bot,
  Context,
  InlineKeyboard,
} from "grammy";

export function registerCallbackHandlers(
  bot: Bot<Context>
): void {
  bot.callbackQuery("option_1", async (ctx) => {
    await ctx.answerCallbackQuery();

    await ctx.editMessageText(
      "You selected Option 1."
    );
  });

  bot.callbackQuery("option_2", async (ctx) => {
    await ctx.answerCallbackQuery();

    await ctx.editMessageText(
      "You selected Option 2."
    );
  });
}

An inline keyboard can be created as follows:

import { InlineKeyboard } from "grammy";

const keyboard = new InlineKeyboard()
  .text("Option 1", "option_1")
  .text("Option 2", "option_2");

await ctx.reply(
  "Choose an option:",
  {
    reply_markup: keyboard,
  }
);

---

4.6 Service Layer

Business logic should be decoupled from Telegram handlers.

Instead of putting database operations directly inside handlers, use services.

"src/services/userService.ts"

import { PrismaClient, User } from "@prisma/client";

export class UserService {
  constructor(
    private readonly db: PrismaClient
  ) {}

  async getOrCreateUser(
    telegramId: bigint,
    username: string | null,
    firstName: string
  ): Promise<User> {
    const existingUser =
      await this.db.user.findUnique({
        where: {
          telegramId,
        },
      });

    if (existingUser) {
      return existingUser;
    }

    return this.db.user.create({
      data: {
        telegramId,
        username,
        firstName,
      },
    });
  }
}

---

5. Database Configuration

Prisma can be used as the ORM.

Prisma Schema

"prisma/schema.prisma"

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id         Int      @id @default(autoincrement())
  telegramId BigInt   @unique
  username   String?
  firstName  String
  state      String   @default("idle")
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
}

For PostgreSQL, change the datasource provider:

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

---

5.1 Prisma Client

"src/models/user.ts"

import { PrismaClient } from "@prisma/client";

export const db = new PrismaClient();

The service can then be initialized using:

import { db } from "../models/user";
import { UserService } from "../services/userService";

export const userService = new UserService(db);

---

6. State Management

For multi-step conversations such as onboarding or form filling, use either a conversation framework or a custom state machine.

Custom State Machine

"src/utils/stateManager.ts"

export enum UserState {
  IDLE = "idle",
  AWAITING_NAME = "awaiting_name",
  AWAITING_EMAIL = "awaiting_email",
}

The current state can be stored in the database.

For example:

await db.user.update({
  where: {
    telegramId,
  },
  data: {
    state: UserState.AWAITING_NAME,
  },
});

Alternatively, temporary session state can be stored in memory.

For production applications, persistent database state is generally preferable when the state needs to survive bot restarts.

---

7. Error Handling

Implement a global error handler to catch exceptions and log them without crashing the bot.

"src/bot/index.ts"

bot.catch(async (error) => {
  console.error(
    "Update caused an error:",
    error.error
  );

  try {
    await error.ctx.reply(
      "An error occurred. Please try again."
    );
  } catch (replyError) {
    console.error(
      "Unable to send error message:",
      replyError
    );
  }
});

The bot continues running while the error is logged.

---

8. Logging

"src/utils/logger.ts"

A simple logger can be implemented using the built-in console:

type LogLevel =
  | "debug"
  | "info"
  | "warn"
  | "error";

const levels: Record<LogLevel, number> = {
  debug: 0,
  info: 1,
  warn: 2,
  error: 3,
};

const configuredLevel: LogLevel =
  (process.env.LOG_LEVEL as LogLevel) ?? "info";

function shouldLog(level: LogLevel): boolean {
  return (
    levels[level] >= levels[configuredLevel]
  );
}

export const logger = {
  debug(message: string, ...args: unknown[]) {
    if (shouldLog("debug")) {
      console.debug(message, ...args);
    }
  },

  info(message: string, ...args: unknown[]) {
    if (shouldLog("info")) {
      console.info(message, ...args);
    }
  },

  warn(message: string, ...args: unknown[]) {
    if (shouldLog("warn")) {
      console.warn(message, ...args);
    }
  },

  error(message: string, ...args: unknown[]) {
    if (shouldLog("error")) {
      console.error(message, ...args);
    }
  },
};

For production systems, a structured logging library such as Pino can be used instead.

---

9. Environment Variables

Create a ".env" file:

BOT_TOKEN=your_telegram_bot_token
DATABASE_URL=file:./bot.db
LOG_LEVEL=info

For PostgreSQL:

BOT_TOKEN=your_telegram_bot_token
DATABASE_URL=postgresql://user:password@localhost:5432/telegram_bot
LOG_LEVEL=info

Never commit ".env" to version control.

".gitignore"

node_modules/
dist/
.env
*.db

---

10. Package Configuration

"package.json"

{
  "name": "telegram-bot-framework",
  "version": "1.0.0",
  "description": "Modular non-AI Telegram bot framework",
  "main": "dist/index.js",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest",
    "test:run": "vitest run",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@prisma/client": "^6.0.0",
    "dotenv": "^16.0.0",
    "grammy": "1.45.1"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "prisma": "^6.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.0.0",
    "vitest": "^3.0.0"
  }
}

---

11. TypeScript Configuration

"tsconfig.json"

{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": [
    "src/**/*.ts"
  ]
}

---

12. Testing

Use Vitest for unit testing.

"tests/handlers.test.ts"

import {
  describe,
  expect,
  it,
  vi,
} from "vitest";

describe("Start handler", () => {
  it("should send a welcome message", async () => {
    const sendMessage = vi.fn();

    const mockContext = {
      from: {
        id: 123,
        first_name: "TestUser",
      },

      chat: {
        id: 456,
      },

      api: {
        sendMessage,
      },
    };

    const user = mockContext.from;
    const chat = mockContext.chat;

    await mockContext.api.sendMessage(
      chat.id,
      `Welcome, ${user.first_name}! This is a non-AI Telegram bot framework.`
    );

    expect(sendMessage).toHaveBeenCalledOnce();

    expect(sendMessage).toHaveBeenCalledWith(
      456,
      "Welcome, TestUser! This is a non-AI Telegram bot framework."
    );
  });
});

For more advanced testing, Telegram API calls can be mocked while testing handlers and services independently.

---

13. Dockerization

Create a "Dockerfile":

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY tsconfig.json ./
COPY prisma ./prisma

RUN npx prisma generate

COPY src ./src

RUN npm run build

CMD ["npm", "start"]

Build the image:

docker build -t telegram-bot .

Run it:

docker run --env-file .env telegram-bot

---

14. Inline Mode

Telegram inline functionality allows users to interact with the bot directly from the message field in any chat.

For example:

@YourBot keyword

The bot can return results that the user can select and send into the current chat.

With grammY, an inline query handler can be registered like this:

import { Bot, Context } from "grammy";

export function registerInlineHandler(
  bot: Bot<Context>
): void {
  bot.on("inline_query", async (ctx) => {
    const query = ctx.inlineQuery.query.trim();

    if (!query) {
      return;
    }

    await ctx.answerInlineQuery([
      {
        type: "article",
        id: "1",
        title: "Example Result",
        description: `Result for: ${query}`,
        input_message_content: {
          message_text:
            `You searched for: ${query}`,
        },
      },
    ]);
  });
}

Inline functionality must first be enabled for the bot through Telegram's bot configuration.

---

15. Deep Linking

Telegram bots can receive parameters through deep links.

A link can contain a parameter such as:

https://t.me/YourBot?start=campaign123

The "/start" command can then inspect the parameter.

With grammY:

bot.command("start", async (ctx) => {
  const payload = ctx.match;

  if (payload) {
    console.log(
      "Start payload:",
      payload
    );
  }

  await ctx.reply(
    "Welcome to the bot!"
  );
});

This can be used for referral systems, campaigns, onboarding flows, and other deterministic workflows.

---

16. Attachment Menu Integration

Telegram also supports bot integration with the attachment menu.

The bot can provide functionality that users access from the attachment menu inside supported Telegram chats.

The actual configuration and availability depend on Telegram's bot platform capabilities and configuration.

The application architecture remains the same:

Telegram
   │
   ▼
Gateway
   │
   ▼
Router
   │
   ▼
Handler
   │
   ▼
Service
   │
   ▼
Database / External API

---

17. Streaming Replies

Bots can provide progressive feedback while processing a request.

For example:

const message = await

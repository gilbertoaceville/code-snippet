# Code Samples Documentation

This document provides explanations and context for the code samples provided.

## Snippet 1: Environment Variables Validation

This snippet validates the presence of required environment variables based on the current environment (development or production).

### Explanation:

```javascript
function createENV(name, developmentRequired, productionRequired) {
  return { name, devRequired: developmentRequired, prodRequired: productionRequired };
}

const REQUIRED_ENVS = [
  createENV("NEXTAUTH_SECRET", true, true),
  createENV("NEXTAUTH_URL", true, true),
];

const isDevelopment = process.env.NODE_ENV === "development";
const key = isDevelopment ? "devRequired" : "prodRequired";

function validateEnvs() {
  const enV = process.env.NODE_ENV || "default (PRODUCTION)";
  console.log(`Env vars script validator is being ran against >>> ${enV} <<< environment vars`);

  const missingEnvs = [];

  for (const requiredVar of REQUIRED_ENVS) {
    const isRequiredInCurrentEnv = requiredVar[String(key)];

    if (!isRequiredInCurrentEnv) continue;

    const isExisting = Object.hasOwnProperty(process.env, requiredVar.name);
    if (!isExisting) missingEnvs.push(requiredVar.name);
  }

  // log missing env vars
  if (missingEnvs.length > 0) {
    console.log("Env vars that are missing: ");

    for (const missingEnv of missingEnvs) console.error(`>> ${missingEnv}`);
    console.log("^ Add these vars to your .env file \n");

    throw new Error("Environment variables aren't correctly configured");
  } else {
    console.log("Environment variables are correctly configured");
  }
}

validateEnvs();

// package.json
  "scripts": {
    "clear-cache": "rimraf .next",
    "dev": "cross-env-shell NODE_ENV=development run-s validate:envs clear-cache start:dev",
    "validate:envs": "ts-node -r dotenv/config src/scripts/envs-validator/validate.ts dotenv_config_path=.env",
  },
```

## Snippet 2: Chaining Multiple Middleware

This snippet demonstrates chaining multiple middleware functions together.

### Explanation:

```typescript
// chaining multiple middleware => middleware.ts
import { NextMiddleware, NextResponse } from "next/server";
import createIntlMiddleware from "next-intl/middleware";

type MiddlewareAccumulator = (middleware: NextMiddleware) => NextMiddleware;

// has to be invoked from a util or helper func
function combineMiddlewares(
  functions: MiddlewareFactory[] = [],
  index = 0
); : NextMiddleware {
  const current = functions[index];
  if (current) {
    const next = combineMiddlewares(functions, index + 1);
    return current(next);
  }
  return () => NextResponse.next();
}

// a sample middleware
export const withLogging: MiddlewareAccumulator = next => {
  return async (req: NextRequest, _next: NextFetchEvent) => {
    console.log("Log some data here", req.nextUrl.pathname);
    return next(request, _next);
  };
};

export const withAuthorization: MiddlewareAccumulator = next => {
  return async (req: NextRequest, _next: NextFetchEvent) => {
    const cookieLocale = req.cookies.get("NEXT_LOCALE");
    const locale = cookieLocale?.value || "en";

    const authenticated = "Logged in";

    if (req.nextUrl.pathname.includes("login")) {
      if (authenticated) {
        return NextResponse.redirect(
          new URL(`/${locale}/admin`, req.nextUrl)
        );
      }
    }

    return next(req, _next);
  };
};

export default combineMiddlewares([withLogging, withAuthorization(createIntlMiddleware({
    locales: ["en", "es"],
    defaultLocale: "en",
  }))]);
```


## Snippet 3: Contentful Export and API Route

This snippet exports content from Contentful and defines an API route.

### Explanation:

```javascript
import path from "path";
import fs from "fs-extra";
import contentfulExport from "contentful-export";
import dotenv from "dotenv";

dotenv.config({
  path: path.join(process.cwd(), "/.env.local"),
});

const SPACE_ID = process.env.CONTENTFUL_SPACE_ID;
const MANAGEMENT_API_TOKEN = process.env.CONTENTFUL_MANAGEMENT_API_ACCESS_TOKEN;
const CONTENTFUL_ENVIRONMENT = process.env.CONTENTFUL_ENVIRONMENT;

if (!SPACE_ID) throw new Error("CONTENTFUL_SPACE_ID token is required.");
if (!MANAGEMENT_API_TOKEN)
  throw new Error("CONTENTFUL_MANAGEMENT_TOKEN token is required.");

const exportsDirPath = "/exports/contentful";
const EXPORT_DIR = path.join(process.cwd(), exportsDirPath);

const options = {
  spaceId: SPACE_ID,
  managementToken: MANAGEMENT_API_TOKEN,
  environmentId: CONTENTFUL_ENVIRONMENT || "portfolio",
  exportDir: EXPORT_DIR,
  contentFile: "contentful-export-portfolio.json",
  saveFile: true,
  skipContentModel: true,
  skipWebhooks: true,
  queryEntries: [
    "content_type=portfolioAbout",
    "content_type=experienceDescriptionItem",
  ],
  queryAssets: ["fields.title=null"],
};

const main = () => {
  fs.ensureDirSync(EXPORT_DIR);

  contentfulExport(options).catch(() => {
    console.log("Oh no! Some errors occurred!");
  });
};

main();

export { main };

// => helper function
export function getDataEntries(data: { entries: Array<any> }, locale = "en-US") {
	return data.entries.reduce((acc: any, entry: Record<string, any>) => {
		const contentType = entry.sys.contentType.sys.id;

		if (contentType === "portfolioAbout") {
			acc.about = {
				name: entry.fields.name[parsedLocale],
				title: entry.fields.title[parsedLocale],
				description: entry.fields.description[locale],
			};
		} else if (contentType === "experienceDescriptionItem") {
			if (!acc.experiences) {
				acc.experiences = [];
			}
			acc.experiences.push(entry.fields.text[locale]);
		}

		return acc;
	}, {});
}

// => /api/chat/route.js
export async function POST(request) {
  try {
    const { messages } = await request.json();

    const { about, experiences } = getDataEntries(contentfulData);

    const dataString = `I have these experiences: ${JSON.stringify(
      experiences
    )}.\n\n`;

    const systemInstruction = {
      role: "system",
      content:
        `You are a helpful assistant for ${about?.name}. Here's some information about them: with title ${about.title} and ${about?.description}.\n\n` +
        dataString +
        "Responses should be technical.",
    };

    messages.unshift(systemInstruction);

    const response = await openai.chat.completions.create({
      model: "gpt-3.5-turbo",
      stream: true,
      messages,
    });

    const stream = OpenAIStream(response);
    return new StreamingTextResponse(stream);
  } catch (error) {
    return new NextResponse(
      error?.message || "Something went haywire",
      {
        status: 500,
      }
    );
  }
}

// package.json
  "scripts": {
    "dev": "run-s contentful:export start:dev",
    "contentful:export": "ts-node ./scripts/contentful/contentful-export.mjs",
  },

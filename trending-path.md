# I built a tool to know when you are trending on GitHub

![ezgif com-video-to-gif (34)](https://github.com/github-20k/blog/assets/100117126/e7cacf0a-e457-4993-b005-bac32655df26)

## TL;DR


If you are a maintainer of a GitHub repository, you might want to get some contributors, stars, and visibility.

The best way to do it is to get into the [GitHub trending feed](https://github.com/trending).
For a small example, at the beginning of October, [Novu](https://github.com/novuhq/novu) was on the trending feed for a week and got more than 4,000 stars in the process.

The problem is that some people don’t know they are on the trending list.

To check out the monitoring tool, [click here](https://gitup.dev/).


![GitTrend](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xod3llt8aig5w9uz9zrs.gif)

&nbsp;

## You need to know when you are trending

You want to add more gas once you are trending, promote it, and stay there longer.

So far, the only way to know you are there is actually to check this feed.
GitHub will not tell you this.

Hell, you can’t even use GitHub GraphQL to check the trending list.

I decided to build a simple solution that can help everybody to know when they are trending.

![wow](https://github.com/github-20k/blog/assets/100117126/2eab31aa-f032-4c3c-9e52-c04f1d6c7e5e)

&nbsp;

## The best technology for this is 🥁

Alright, so it might not be the best, but it’s the best for me.

- I am a React person, so [NextJS](https://nextjs.org/) is an obvious solution for me.
It gives me both frontend and backend. It auto-scales it for me in the cloud without the need to manually add more containers **(when using Vercel)**.
&nbsp;

- To store everything in my database, I decided to go with Postgres and [Prisma](https://www.prisma.io).
For our demo, we will use SQLite.
&nbsp;

- I didn’t want to create another background job instance and take care of scaling it (cron + queue). I prefer to stick with the Vercel deployment (it’s also free). For that, I have chosen [Trigger.dev](http://Trigger.dev). It allows me to create background jobs inside NextJS and so much more, such as monitoring and logging.
&nbsp;

- I needed to send notifications to people who were trending. For that, I chose [Novu](https://novu.co). You probably think that’s overkill, but actually, I couldn’t have done it without Novu, and you will find out why later.
&nbsp;


![Cover](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/htxkp3abn42qesj1cghp.png)

<hr />

## How to build this thing 🤔

Set up a new project with NextJS

```
npx create-next-app@latest
```

After that, add Prisma

```
npm install prisma @prisma/client --save
npx prisma init --datasource-provider sqlite
```

Add other libraries we are going to use:

```
npm install axios jsdom @types/axios @types/jsdom --save
```

I don’t use fetch. I love Axios 😻

Edit the created `schema.prisma` and add the following code:

```
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Account {
  id                 String  @id @default(cuid())
  userId             String
  type               String
  provider           String
  providerAccountId  String
  refresh_token      String?  @db.Text
  access_token       String?  @db.Text
  expires_at         Int?
  token_type         String?
  scope              String?
  id_token           String?  @db.Text
  session_state      String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  handle        String?   @unique
  emailVerified DateTime?
  image         String?
  userRepo      UserRepository[]
  accounts      Account[]
  sessions      Session[]
}

model UserRepository {
  id            String    @id @default(cuid())
  repositoryId  Int
  userId        String
  user          User      @relation(fields: [userId], references: [id])
  repository    Repositories @relation(fields: [repositoryId], references: [id])
  @@unique([repositoryId, userId])
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

model Repositories {
  id              Int      @id @default(autoincrement())
  url             String
  language        String
  history         RepositoriesHistory[]
  userRepo        UserRepository[]
  languagePlace   Int
  trendingPlace   Int
  updatedAt       DateTime @default(now())
  @@unique([url])
  @@index([url])
  @@index([language])
  @@index([languagePlace])
  @@index([trendingPlace])
}

model RepositoriesHistory {
  id              Int      @id @default(autoincrement())
  repositoryId    Int
  place           Int
  language        String?
  repository      Repositories @relation(fields: [repositoryId], references: [id])
  createdAt       DateTime @default(now())
  @@index([place])
  @@index([language])
}
```

Let’s take a look at what’s going on here.

- **Account, Session, and User, VerificationToken** are required fields when working with [NextAuth](https://next-auth.js.org/) for authentication/authorization. It saves the user's information, token, oAuth (if you implement it), etc.
&nbsp;

- **Repositories** is the table with all the repositories the user will add in the future
    1. url - the full URL of the repository.
    2. language - the primary language of the repository (for example, Novu is typescript, it can trend on the typescript feed)
    3. languagePlace - last known position on the specific language trending feed.
    4. trendingPlace - last known position on the main trending feed.
    5. updatedAt - last time we updated it.
&nbsp;

- **RepositoriesHistory** is the table to save all the past trends to know what happened before.

Once done, we can run `npx prisma db push` to update our database with the new tables. 

<hr />

## Next steps 🚶🏻‍♂️

I will not take you on creating the React infrastructure and logging in.
I have talked about it a lot in my other [63 posts](https://dev.to/dashboard).
You can also read the NextAuth [quick start guide](https://next-auth.js.org/getting-started/example).
For now, let’s assume the login is completed.
The person is logged in to the system.
Here are our next steps:
- We want the person to add their repositories
- We want to register this person (and all the other people) who registered to get updates for that repository to Novu so we can later tell all of them the repository is trending.
- We want to create a background process that checks for trending repositories.
- We want to inform people that they are trending.

---

## Set up Notifications

1. Go ahead and register for [Novu Cloud](https://web.novu.co/auth/signup)
2. Head over to settings and copy the API Key and Application Identifier to your `.env.local` file


![Novu](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4lgbyp71aff02jyr5ewq.png)

```bash
NEXT_PUBLIC_NOVU_APP_ID=
NOVU_SECRET=
```

Head over the workflows and create a new workflow called trending.
The workflow should have three steps.

- **Digest** - If users are trending for multiple repositories or lists, we don’t want to spam them. We can merge everything into a single notification with the Digest. Set the Digest for 5 minutes. The job shouldn’t take long.
- **In-App** - We want to add a notification in the user “bell icon” on the dashboard - it might not be so usable, but it’s good to see the history.
- **Email** - We want to inform users that they are trending over email.

If you are building a mobile app for that, add something like a push notification.


![Novu2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kilqsptg36gomlwlj1k1.png)

Inside each step, you want to add the notification with the digest message, something like

```
{{#if step.digest}}
  {{#each step.events}}
    {{text}}
  {{/each}}
{{else}}
  {{text}}
{{/if}}
```

The `{{text}}` later can be something like “You are trending for clickvote on place 8”

Create a folder inside of `src` called `helpers`, create a new file called `novu.ts`, and add the following code:

```jsx
import {Novu} from "@novu/node";

export const novu = new Novu(process.env.NOVU_SECRET!);
```

We have to identify the users before we send them notifications.
For that, go to your NextAuth code and add `events` to `NextAuthOptions` like this:

```javascript
events: {
    async signIn({ user, account }) {
        await novu.subscribers.identify(user.email!, {
            email: user.email!
        });
    }
}
```

Create a new file inside of helpers called `all.languages.ts`

This is basically a pre-work I have done to add all the GitHub languages and their menu slug.

It’s a huge file, so copy it from:
https://github.com/github-20k/trending-list/blob/main/src/helpers/all.languages.ts

Now, let’s create a new API endpoint to add new repositories `/api/add`

```jsx
import type { NextApiRequest, NextApiResponse } from 'next'
import {prisma} from "../../../prisma/prisma";
import axios from "axios";
import {nextOptions} from "@trending/pages/api/auth/[...nextauth]";
import {getServerSession} from "next-auth/next";
import {allLanguages} from "@trending/helpers/all.languages";
import {novu} from "@trending/helpers/novu";

export const extractGithubInfo = (url: string) => {
  const regex = /https?:\/\/github\.com\/([^\/]+)\/([^\/]+)/;
  const match = url.match(regex);

  if (match) {
    return {
      owner: match[1],
      name: match[2]
    };
  } else {
    return false;
  }
}
const getLanguages = async (url: string, token: string) => {
  const extract = extractGithubInfo(url);
  if (!extract) return false;

  try {
    const {data} = await axios.get(`https://api.github.com/repos/${extract.owner}/${extract.name}/languages`, {
      withCredentials: true
    });

    const findLanguage = Object.keys(data).reduce((all, current) => {
      if (data[current] > all) {
        return data[current];
      }
      return all;
    });

    const slug = allLanguages.find(p => p.name.toLowerCase() === findLanguage.toLowerCase());
    if (!slug?.slug) {
      return false;
    }
    return slug?.slug;
  }
  catch (err) {
    return false;
  }
}

const createRepository = async (repository: string, language: string) => {
  try {
    const create = await prisma.repositories.create({
      data: {
        url: repository, language: language as string, languagePlace: 0, trendingPlace: 0
      }
    });

    await novu.topics.create({
      name: 'notifications for repository',
      key: `repository:${create.id}`
    });

    return create;
  }
  catch (err) {
    return false;
  }
}

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST' || !req?.body?.repository || !req.body.repository.match(/^https:\/\/github\.com\/[^/]+\/[^/]+\/?$/)) {
    res.status(200).json({ valid: false });
    return ;
  }

  if (req.body.repository.at(-1) === '/') {
    req.body.repository = req.body.repository.slice(0, -1);
  }

  const session = await getServerSession(req, res, nextOptions);
  if (!session?.user) {
    res.status(200).json({ valid: false });
    return ;
  }
  // @ts-ignore
  const language = await getLanguages(req.body.repository, session.user.access_token);
  if (!language) {
    res.status(200).json({ valid: false });
  }

  const repository = await createRepository(req.body.repository, language as string) || await prisma.repositories.findFirst({
    where: {
      url: req.body.repository
    }
  });

  try {
    await prisma.userRepository.create({
      data: {
        // @ts-ignore
        userId: session.user.id as string, // @ts-ignore
        repositoryId: repository.id as number
      }
    });

    await novu.topics.addSubscribers(`repository:${repository?.id!}`, {
      // @ts-ignore
      subscribers: [session.user.email]
    });
  }
  catch (err) {
    res.status(200).json({ valid: false });
  }

  res.status(200).json({ valid: true })
}
```

**Let’s see what’s going on here:**

- We get a new request and extract the `owner` and the `name` from the GitHub URL, for example, https://github.com/novuhq/novu (owner is `novuhq`, and the name is `novu`).
- Then we go to GitHub to check that the repository exists and get the primary language of the repository, for example, `typescript`
- We insert the new repository into the `Repositories` table. If we succeed, we will create a new [topic](https://docs.novu.co/subscribers/topics) inside Novu with the ID of the repository from the DB.
Later, we can tell Novu to notify about trending to everyone on that topic.
- If the repository already exists, it just takes the existing repository from our table.
- It adds a connection between the repository and the user (so we can see it on the dashboard).
- It adds the user to the topic, so later, we can send them a notification about this repository.

Now, we can create a route to delete a registration to that repository `/api/remove`

```jsx
import type { NextApiRequest, NextApiResponse } from 'next'
import {nextOptions} from "@trending/pages/api/auth/[...nextauth]";
import {getServerSession} from "next-auth/next";
import {prisma} from "../../../prisma/prisma";
import {novu} from "@trending/helpers/novu";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST' || !req?.body?.repository || !req.body.repository.match(/^https:\/\/github\.com\/[^/]+\/[^/]+\/?$/)) {
    res.status(200).json({ valid: false });
    return ;
  }

  if (req.body.repository.at(-1) === '/') {
    req.body.repository = req.body.repository.slice(0, -1);
  }

  const session = await getServerSession(req, res, nextOptions);
  if (!session?.user) {
    res.status(200).json({ valid: false });
    return ;
  }

  const repository = await prisma.repositories.findFirst({
    where: {
      url: req.body.repository
    }
  });

  await prisma.userRepository.deleteMany({
    // @ts-ignore
    where: {
      repositoryId: repository?.id! as number,
      // @ts-ignore
      userId: session?.user?.id!
    }
  });

  await novu.topics.removeSubscribers(`repository:${repository?.id!}`, {
    // @ts-ignore
    subscribers: [session.user.email]
  });

  res.status(200).json({ valid: true })
}
```

We are basically removing the connection of the user from the repository.
We also remove the user from subscribing to that topic in Novu, but we don’t remove the repository. 

If you want to do some extra work, you can check if there are no subscribers in the repository, remove the repository from the database, and delete the topic.

---

## Let’s set up some background work

We need to set up [Trigger.dev](http://Trigger.dev) on our project.

On our main root project, run `npx @trigger.dev/cli@latest init`.
It will take care of everything.

If you need help setting everything up, check out their [quick start guide](https://trigger.dev/docs/documentation/quickstarts/nextjs). 

Once done, you will see a few new files and folders created:

1. `/api/trigger.ts` - This is the API call [trigger.dev](http://trigger.dev) call from their side (never touch this) 
2. jobs folder, where you can define different jobs like crons and queues.

Let’s create a new cron job called “Check trending” that will run every hour.
Create a new file called `check-trending.ts` and add the following code:

```jsx
import { Job, cronTrigger } from "@trigger.dev/sdk";
import {client} from "@trending/trigger";
import {prisma} from "../../prisma/prisma";

client.defineJob({
  id: "check-trending",
  name: "Check trending",
  version: "0.0.1",
  trigger: cronTrigger({
    cron: "0 * * * *",
  }),
  run: async (payload, io, ctx) => {
    const repositories = await prisma.repositories.findMany({
      select: {
        language: true,
      },
      distinct: ["language"],
    });

    for (const repository of [{language: ''}, ...repositories]) {
      await io.logger.info("trigger for " + repository.language);
      await io.sendEvent('process-language-' + repository.language, {
        name: "process.language",
        payload: {
            language: repository.language,
        }
      });
    }

    await io.logger.info("repo", {
      repositories,
    });

    return { repositories };
  },
});
```

This job will run every hour.

It will go to our database and take all the languages that we have, for example, `typescript`, `python`, `go`, etc.

Each language will be sent to a different queue to scrape that specific language trending feed.

> You can see that we send one empty language that’s for the main trending feed.

Alright, now let’s work on processing each language feed.
Create a new file called `process-language.ts`

**Here is the code:**

```jsx
import {eventTrigger} from "@trigger.dev/sdk";
import {client} from "@trending/trigger";
import {z} from "zod";
import axios from "axios";
import { JSDOM } from 'jsdom';
import {prisma} from "../../prisma/prisma";

client.defineJob({
  id: "process-language",
  name: "Process language",
  version: "0.1.0",
  trigger: eventTrigger({
    name: "process.language",
    schema: z.object({
      language: z.string(),
    }),
  }),
  //this function is run when the custom event is received
  run: async (payload, io, ctx) => {
    const {data} = await axios.get(`https://github.com/trending/${payload.language.replace('#', '%23')}`);
    const dom = new JSDOM(data);

    const list = Array.from(dom.window.document.querySelectorAll('article h2')).map((p, index) => ({
        rank: index + 1,
        name: p?.textContent?.replace(/\s+/g, ' ').trim().split('/').map(p => p.trim()).join('/'),
    }));

    const foundRepositories = await prisma.repositories.findMany({
        where: {
            ...(payload.language === '' ? {} : {language: payload.language}),
            url: {
                in: list.map(p => 'https://github.com/' + p.name),
            }
        }
    });

    for (const repository of foundRepositories) {
        const findRank = list.find(p => 'https://github.com/' + p.name === repository.url);
        if (
            (payload.language === '' && repository.trendingPlace !== findRank?.rank) ||
            (payload.language !== '' && repository.languagePlace !== findRank?.rank)
        ) {
            await io.sendEvent('update-position-' + findRank?.name?.replace('/', '-'), {
                name: 'update.position',
                payload: {
                    link: repository.url,
                    rank: findRank?.rank!,
                    language: payload.language,
                }
            });
        }
    }

    await io.sendEvent('reset-position-' + payload.language, {
      name: 'reset.positions',
      payload: {
          links: foundRepositories.map(p => p.url),
          language: payload.language,
      }
    });

    return payload;
  },
});
```

We send an HTTP request to `https://github.com/trending/{language}` to check all the trending repositories in that language.

Since we are “scraping” the page, we need to convert the HTML into Javascript.
To parse the content of the page, I have used `jsdom`.

Then, we query the database for all those repositories in that specific language.

We iterate and check if the repository changed position.
If it did, we send it to a new queue called `update.position`

Then, we send a reset position event (`reset.positions`) for all those that are not on the trending feed (to achieve that, we need to send the ones that are on the feed, and in the query, we will ask for all the repositories that are not on 0 positions and not one of those repositories)

Now, let’s create the `update.position` job.

Create a new file called: `update-position.ts`

```jsx
import {eventTrigger} from "@trigger.dev/sdk";
import {client} from "@trending/trigger";
import {z} from "zod";
import {prisma} from "../../prisma/prisma";
import {novu} from "@trending/helpers/novu";
import {TriggerRecipientsTypeEnum} from "@novu/shared";
import {extractGithubInfo} from "@trending/pages/api/add";

const buildMessage = (link: string, newRank: number, oldRank: number, language: string) => {
    const extract = extractGithubInfo(link);
    if (!extract) {
        return '';
    }

    if (oldRank === 0) {
        return language ?
            `Wow! ${extract.owner}/${extract.name} is now trending for ${language} on place ${newRank}` :
            `OMG! ${extract.owner}/${extract.name} is now trending on the main feed on place ${newRank}`;
    }
    else if (oldRank > newRank) {
        return language ?
            `Yay! ${extract.owner}/${extract.name} bumped from place ${oldRank} to place ${newRank} on ${language}` :
            `Super! ${extract.owner}/${extract.name} bumped from place ${oldRank} to place ${newRank} on the main feed`;
    }
    else if (newRank > oldRank) {
        return language ?
            `Bummer! ${extract.owner}/${extract.name} downgraded from place ${oldRank} to place ${newRank} on ${language}` :
            `Damn! ${extract.owner}/${extract.name} downgraded from place ${oldRank} to place ${newRank} on the main feed`;
    }
}

client.defineJob({
  id: "update-position",
  name: "Update position",
  version: "0.1.0",
  trigger: eventTrigger({
    name: "update.position",
    schema: z.object({
      link: z.string(),
      language: z.string(),
      rank: z.number(),
    }),
  }),
  //this function is run when the custom event is received
  run: async (payload, io, ctx) => {
      const find = await prisma.repositories.findFirst({
        where: {
          url: payload.link,
        }
      });

      if (!find) {
        return ;
      }

      await prisma.repositories.updateMany({
        where: {
          id: find.id,
        },
        data: payload.language === '' ? {
          trendingPlace: payload.rank
        } : {
          languagePlace: payload.rank
        }
      });

      await prisma.repositoriesHistory.create({
        data: {
          repositoryId: find.id,
          language: payload.language,
          place: payload.rank
        }
      });

      const message = buildMessage(payload.link, payload.rank, find.language ? find.languagePlace : find.trendingPlace, payload.language);

      if (!message) {
          return ;
      }

      await novu.trigger('trending', {
          to: [{
              type: TriggerRecipientsTypeEnum.TOPIC,
              topicKey: `repository:${find.id}`
          }],
          payload: {
              text: message,
          }
      });
  },
});
```

We first take the repository from the database by the repository name.

Then, we update our database with the new value of the trending position.
We add new value to our history table. It’s always good to know what happened in the past.

We build the message we want to send to the user. It can be any of the following:

- Trending for a specific language higher position
- Trending on the main feed higher position
- Trending for a specific language lower position
- Trending on the main feed lower position

Then we use Novu to send events to all the registered people to that repository ID (cool, right?)

It will trigger all the workflow, including the **Digest, In-App, and Email.**

Now, the last thing we want is to let people know their trend is finished.

For that, we will create a new job called `reset.positions`

Create a new file called `reset-position.ts`. Here is the full code:

```jsx
import {eventTrigger} from "@trigger.dev/sdk";
import {client} from "@trending/trigger";
import {z} from "zod";
import {prisma} from "../../prisma/prisma";
import {novu} from "@trending/helpers/novu";
import {TriggerRecipientsTypeEnum} from "@novu/shared";
import {extractGithubInfo} from "@trending/pages/api/add";

client.defineJob({
  id: "Reset positions",
  name: "Reset positions",
  version: "0.1.0",
  trigger: eventTrigger({
    name: "reset.positions",
    schema: z.object({
      links: z.array(z.string()),
      language: z.string(),
    }),
  }),
  //this function is run when the custom event is received
  run: async (payload, io, ctx) => {
      const findMany = await prisma.repositories.findMany({
          where: {
              url: {
                  notIn: payload.links,
              },
              ...(payload.language === '' ? {} : {language: payload.language}),
              ...payload.language === '' ? {
                  trendingPlace: {
                      gt: 0
                  }
              } : {
                  languagePlace: {
                      gt: 0
                  }
              }
          }
      });

      for (const repo of findMany) {
          const extract = extractGithubInfo(repo.url);
          if (!extract) {
              continue;
          }

          await prisma.repositories.update({
              where: {
                  id: repo.id
              },
              data: payload.language === '' ? {
                  trendingPlace: 0
              } : {
                  languagePlace: 0
              }
          });

          await prisma.repositoriesHistory.create({
              data: {
                  place: 0,
                  language: payload.language,
                  repositoryId: repo.id
              }
          });

          await novu.trigger('trending', {
              to: [{
                  type: TriggerRecipientsTypeEnum.TOPIC,
                  topicKey: `repository:${repo.id}`
              }],
              payload: {
                  text: payload.language ?
                      `That was a good run! ${extract.owner}/${extract.name} is not trending for ${repo.language} anymore` :
                      `Nice run! ${extract.owner}/${extract.name} is not trending on the main feed anymore`
              }
          });
      }
  },
});
```

- We find all the repositories places that are higher than 0 but are not on the trending feed.
- We update our database with the new position.
- We add it to our trending history.
- We send everybody registered to this repository that they are not trending anymore with Novu.

Now edit your `index.ts` file inside of the job and add the following code:

```jsx
//Export all your job files here
export * from "./check-trending";
export * from "./process-language";
export * from "./update-position";
export * from "./reset-position";
```

To run all the jobs locally, open a new terminal and run `npx @trigger.dev/cli@latest dev`

It’s super cool, and it uses ngrok to make your path public so they can send you a request.

In production, [you can use this deployment tutorial](https://trigger.dev/docs/documentation/guides/deployment-setup)

And you are done! 🥳

<hr />

If you want to monitor your repository (or somebody else repository) for trending, feel free to use this link here: https://gitup.dev

If you want to self-host it yourself, you can clone this repository:
https://github.com/github-20k/trending-list

If you enjoyed this article, make sure you:

Star the Novu repository

Star the [Trigger.dev](http://Trigger.dev) repository

See you next time 😎

<hr />

Follow me on X.
I share some nice nuggets about open-source growth:
https://twitter.com/nevodavid

![In for tech](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/86ywkzncq6erg44d6xy9.gif)
